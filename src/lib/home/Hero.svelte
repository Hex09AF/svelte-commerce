<script>
import { onMount } from 'svelte'
import { SplideSlide } from '@splidejs/svelte-splide'
import LazyImg from '$lib/components/Image/LazyImg.svelte'

export let banners = []

$: sliderBanners = banners?.filter((b) => {
	return b.type === 'slider' && b.isMobile === false
})

$: sliderBannersMobile = banners?.filter((b) => {
	return b.type === 'slider' && b.isMobile === true
})

let Carousel, Splide

onMount(async () => {
	const SplideModule = await import('$lib/components/SplideJs.svelte')
	Splide = SplideModule.default
})
</script>

<!-- <div class="w-full overflow-hidden">
	{#if sliderBanners?.length > 0}
		<LazyImg
			src="{sliderBanners[0]?.img}"
			alt="banner"
			width="1500"
			height="380"
			class="h-full w-full object-contain object-center" />
	{:else}
		<div class="h-full w-full animate-pulse bg-gray-300"></div>
	{/if}
</div> -->

{#if sliderBanners?.length > 0}
	<div class="relative mx-auto hidden h-auto w-full overflow-hidden bg-white sm:block">
		<svelte:component this="{Splide}" options="{{ type: 'loop', autoplay: true, lazyLoad: true }}">
			{#each sliderBanners as b, ix}
				{#if b.img}
					<SplideSlide>
						<a
							href="{b.link || '##'}"
							aria-label="Click to route into banner related products page"
							data-sveltekit-preload-data
						>
							<LazyImg
								src="{b.img}"
								alt="{b.name}"
								height="380"
								aspect_ratio="4:1"
								class="block h-auto w-full object-contain object-top"
							/>
						</a>
					</SplideSlide>
				{/if}
			{/each}
		</svelte:component>

		<!-- <svelte:component this="{Carousel}">
			{#each sliderBanners as b, ix}
				{#if b.img}
					<a
						href="{b.link || '##'}"
						aria-label="Click to route into banner related products page"
						data-sveltekit-preload-data
						class="carousel-item relative  float-left h-auto w-full {ix == 0 ? 'active' : ''}">
						<LazyImg
							src="{b.img}"
							alt="{b.name}"
							height="380"
							aspect_ratio="4:1"
							class="block h-auto w-full object-contain object-top" />
					</a>
				{/if}
			{/each}
		</svelte:component> -->
	</div>
{/if}

{#if sliderBannersMobile?.length > 0}
	<div class="mx-auto block h-auto w-full overflow-hidden bg-white sm:hidden">
		<svelte:component this="{Carousel}">
			{#each sliderBannersMobile as b, ix}
				{#if b.img}
					<a
						href="{b.link || '##'}"
						aria-label="Click to route into banner related products page"
						data-sveltekit-preload-data
						class="carousel-item relative float-left h-auto w-full {ix == 0 ? 'active' : ''}"
					>
						<LazyImg
							src="{b.img}"
							alt="{b.name}"
							height="190"
							aspect_ratio="4:1"
							class="block h-auto w-full object-contain object-top"
						/>
					</a>
				{/if}
			{/each}
		</svelte:component>
	</div>
{/if}
