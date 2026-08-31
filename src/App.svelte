<script>
  import { onMount } from 'svelte';

  let status = 'loading'; // loading | ready | error
  let errorMessage = '';
  let imageUrl = '';
  let pageUrl = '';
  let artist = '';
  let licenseName = '';
  let placeName = '';
  let lat = null;
  let lon = null;

  let prefetchedPhoto = null;
  let isPrefetching = false;
  let requestId = 0;

  const locationCategories = [
    'Landscape photography',
    'Nature photography',
    'Cityscape photography',
    'Architecture photography',
    'Mountains',
    'Lakes',
    'Waterfalls',
    'Beaches',
    'Forests',
    'National parks',
    'Natural landscapes',
    'Rivers',
    'Deserts',
    'Islands',
    'Villages',
    'Cities'
  ];

  const blockedTerms = [
    'portrait',
    'person',
    'people',
    'man',
    'women',
    'woman',
    'men',
    'child',
    'children',
    'boy',
    'girl',
    'face',
    'faces',
    'selfie',
    'people photography',
    'portrait photography',
    'group photo',
    'family',
    'personnel'
  ];

  function stripHtml(html) {
    if (!html) return '';

    const div = document.createElement('div');
    div.innerHTML = html;

    return (div.textContent || '').trim();
  }

  function looksHumanCentric(page) {
    const title = (page.title || '').toLowerCase();

    const categories = (page.categories || [])
      .map((category) => category.title || '')
      .join(' ')
      .toLowerCase();

    const text = `${title} ${categories}`;

    return blockedTerms.some((term) => text.includes(term));
  }

  async function fetchCategory(category) {
    const endpoint =
      'https://commons.wikimedia.org/w/api.php' +
      '?action=query' +
      '&generator=categorymembers' +
      `&gcmtitle=${encodeURIComponent(`Category:${category}`)}` +
      '&gcmtype=file' +
      '&gcmlimit=50' +
      '&prop=imageinfo|coordinates|categories' +
      '&iiprop=url|extmetadata' +
      '&iiurlwidth=1600' +
      '&cllimit=20' +
      '&format=json' +
      '&origin=*';

    const res = await fetch(endpoint);

    if (!res.ok) {
      throw new Error('Commons request failed');
    }

    const data = await res.json();

    return Object.values(data?.query?.pages || {});
  }

  async function getRandomPhoto() {
    /*
     * Shuffle the categories and try only one or two.
     * This keeps API usage low while still giving variety.
     */
    const categories = [...locationCategories].sort(
      () => Math.random() - 0.5
    );

    for (const category of categories.slice(0, 2)) {
      try {
        const pages = await fetchCategory(category);

        const candidates = pages.filter((page) => {
          return (
            page.imageinfo?.[0] &&
            page.coordinates?.[0] &&
            !looksHumanCentric(page)
          );
        });

        if (candidates.length === 0) {
          continue;
        }

        const page =
          candidates[
            Math.floor(Math.random() * candidates.length)
          ];

        const info = page.imageinfo[0];
        const metadata = info.extmetadata || {};
        const coordinate = page.coordinates[0];

        return {
          imageUrl: info.thumburl || info.url,
          pageUrl: info.descriptionurl,
          artist:
            stripHtml(metadata.Artist?.value) ||
            'Unknown photographer',
          licenseName:
            metadata.LicenseShortName?.value || '',
          lat: coordinate.lat,
          lon: coordinate.lon
        };
      } catch {
        // Try the next category.
      }
    }

    throw new Error(
      'No suitable landscape photos came back. Try again.'
    );
  }

  function preloadImage(url) {
    return new Promise((resolve, reject) => {
      const img = new Image();

      img.onload = resolve;
      img.onerror = reject;
      img.src = url;
    });
  }

  async function reverseGeocode(latitude, longitude, id) {
    try {
      const res = await fetch(
        `https://nominatim.openstreetmap.org/reverse?format=jsonv2&lat=${latitude}&lon=${longitude}&zoom=10&accept-language=en`
      );

      if (!res.ok) {
        throw new Error('reverse geocode failed');
      }

      const data = await res.json();

      /*
       * Don't allow an old request to overwrite the
       * location of a newer wallpaper.
       */
      if (id !== requestId) {
        return;
      }

      const address = data.address || {};

      const locality =
        address.city ||
        address.town ||
        address.village ||
        address.hamlet ||
        address.county ||
        address.state ||
        address.region;

      const country = address.country;

      placeName = [locality, country]
        .filter(Boolean)
        .join(', ');

      if (!placeName) {
        placeName =
          data.display_name ||
          'Somewhere on Earth';
      }
    } catch {
      if (id === requestId) {
        placeName = 'Somewhere on Earth';
      }
    }
  }

  async function prefetchNext() {
    if (isPrefetching || prefetchedPhoto) {
      return;
    }

    isPrefetching = true;

    try {
      const photo = await getRandomPhoto();

      /*
       * Preload the actual image while the user is
       * looking at the current one.
       */
      await preloadImage(photo.imageUrl);

      prefetchedPhoto = photo;
    } catch {
      // Prefetch failure is intentionally silent.
    } finally {
      isPrefetching = false;
    }
  }

  async function showPhoto(photo) {
    const id = ++requestId;

    imageUrl = photo.imageUrl;
    pageUrl = photo.pageUrl;
    artist = photo.artist;
    licenseName = photo.licenseName;
    lat = photo.lat;
    lon = photo.lon;

    placeName = '';
    status = 'ready';

    /*
     * Location lookup happens after the image is already
     * visible, so it doesn't block rendering.
     */
    reverseGeocode(lat, lon, id);

    /*
     * Start preparing the next photo after this one
     * is visible.
     */
    prefetchNext();
  }

  async function fetchRandomPhoto() {
    status = 'loading';
    errorMessage = '';

    try {
      /*
       * If we already prepared an image in the background,
       * use it immediately.
       */
      if (prefetchedPhoto) {
        const photo = prefetchedPhoto;

        prefetchedPhoto = null;

        await showPhoto(photo);

        return;
      }

      const photo = await getRandomPhoto();

      await showPhoto(photo);
    } catch (error) {
      errorMessage =
        error?.message ||
        'Something went wrong';

      status = 'error';
    }
  }

  onMount(() => {
    fetchRandomPhoto();
  });
</script>

<main
  class="stage"
  class:is-ready={status === 'ready'}
  style={status === 'ready'
    ? `background-image: url("${imageUrl}")`
    : ''}
>
  {#if status === 'loading'}
    <div class="state-message">
      <span class="dot"></span>
      Finding somewhere on Earth&hellip;
    </div>
  {:else if status === 'error'}
    <div class="state-message">
      <span>{errorMessage}</span>

      <button
        class="retry"
        onclick={fetchRandomPhoto}
      >
        Try another
      </button>
    </div>
  {/if}

  {#if status === 'ready'}
    <a
      class="info"
      href={pageUrl}
      target="_blank"
      rel="noopener noreferrer"
      aria-label={`Open photo source on Wikimedia Commons: ${placeName}`}
    >
      <span class="place">
        {placeName || 'Locating…'}
      </span>

      <span class="separator">·</span>

      <span>Photo by {artist}</span>
    </a>
  {/if}
</main>

<style>
  :global(html, body) {
    margin: 0;
    padding: 0;
    height: 100%;
    background: #0d0d0d;
  }

  :global(*) {
    box-sizing: border-box;
  }

  .stage {
    position: relative;

    width: 100vw;
    height: 100vh;

    background-color: #0d0d0d;
    background-size: cover;
    background-position: center;

    display: flex;
    align-items: flex-end;
    justify-content: flex-end;

    color:#666;

    padding: clamp(16px, 4vw, 40px);

    opacity: 0;

    transition: opacity 0.6s ease;

    font-family:
      -apple-system,
      BlinkMacSystemFont,
      'Segoe UI',
      Helvetica,
      Arial,
      sans-serif;
  }

  .stage.is-ready {
    opacity: 1;
  }

  .state-message {
    margin: auto;

    color: #d8d8d8;

    font-size: 15px;

    display: flex;
    align-items: center;
    gap: 10px;
  }

  .dot {
    width: 7px;
    height: 7px;

    border-radius: 50%;

    background: #d8d8d8;

    animation: pulse 1.1s ease-in-out infinite;
  }

  @keyframes pulse {
    0%,
    100% {
      opacity: 0.25;
    }

    50% {
      opacity: 1;
    }
  }

  .retry {
    background: transparent;

    border: 1px solid #666;
    color: #d8d8d8;

    border-radius: 999px;

    padding: 6px 14px;

    font-size: 13px;

    cursor: pointer;

    transition:
      border-color 0.2s ease,
      color 0.2s ease;
  }

  .retry:hover {
    border-color: #d8d8d8;
    color: #fff;
  }

  .retry:focus-visible {
    outline: 2px solid #fff;
    outline-offset: 3px;
  }

  .info {
    position: absolute;

    right: 16px;
    bottom: 16px;

    display: flex;
    align-items: center;
    gap: 7px;

    max-width: calc(100vw - 32px);
    background: rgba(0, 0, 0, 0.2);
    backdrop-filter: blur(4px);
    padding: 6px 12px;
    color: rgba(255, 255, 255, 0.78);
    border-radius: 5px;

    font-size: 12px;
    line-height: 1.4;

    text-decoration: none;

    transition:
      color 0.2s ease,
      opacity 0.2s ease;
  }

  .info:hover {
    color: #fff;
    background: rgba(0, 0, 0, 0.3);
  }

  .info:focus-visible {
    outline: 2px solid #fff;
    outline-offset: 4px;
    border-radius: 3px;
  }

  .info .place {
    font-size: 12px;
    color: #fff;
  }

  .info .separator {
    opacity: 0.45;
  }

  @media (max-width: 700px) {
    .info {
      right: 16px;
      bottom: 16px;

      max-width: calc(100vw - 32px);

      font-size: 11px;
    }

    .info .place {
      font-size: 13px;
    }
  }

  @media (max-width: 500px) {
    .info {
      flex-wrap: wrap;
      justify-content: flex-end;
    }
  }

  @media (prefers-reduced-motion: reduce) {
    .stage,
    .info,
    .retry,
    .dot {
      transition: none;
      animation: none;
    }
  }
</style>