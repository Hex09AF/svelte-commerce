<style>
@media (min-width: 1024px) {
	.top-rem-map {
		top: 85px;
	}
}
.frosted-black {
	backdrop-filter: blur(5px);
	background-color: hsla(0, 0%, 0%, 0.8);
}
.height {
	max-height: 85vh;
}
</style>

<script>
import { currency, dateOnly, toast } from '$lib/utils'
import { fade } from 'svelte/transition'
import { ProductService, PopularSearchService } from '$lib/services'
import { goto, invalidateAll } from '$app/navigation'
import { onMount } from 'svelte'
import { page } from '$app/stores'
import { sorts } from '$lib/config'
import CatelogNav from '$lib/CatelogNav.svelte'
import dayjs from 'dayjs'
import DesktopFilter from '$lib/components/DesktopFilter.svelte'
import dotsLoading from '$lib/assets/dots-loading.gif'
import DummyProductCard from '$lib/DummyProductCard.svelte'
import MobileFilter from '$lib/components/MobileFilter.svelte'
import noDataAvailable from '$lib/assets/no/no-data-available.png'
import Pagination from '$lib/components/Pagination.svelte'
import PrimaryButton from '$lib/ui/PrimaryButton.svelte'
import ProductCard from '$lib/ProductCard.svelte'
import SEO from '$lib/components/SEO/index.svelte'

export let data

// console.log('data = ', data)
// console.log('data = ', data.category)
// console.log('Products = ', products)
// console.log('Count = ', count)
// console.log('Facets = ', facets)

let today = dayjs(new Date()).toISOString()

let seoProps = {
	// addressCountry: 'India',
	// addressLocality: 'Semiliguda, Koraput',
	// addressRegion: 'Odisha',
	// alternateJsonHref: '',
	// alternateXml: { title: '', href: '' },
	brand: $page.data.store?.title,
	breadcrumbs: data.category?.children,
	caption: $page.data.store?.title,
	category: data.searchData,
	contentUrl: $page.data.store?.logo,
	createdAt: today,
	// depth: { unitCode: '', value: '' },
	email: `${$page?.data?.store?.email}`,
	// entityMeta: '',
	// facebookPage: '',
	// gtin: '',
	// height: '',
	id: $page?.url?.href,
	image: $page.data.store?.logo,
	logo: $page.data.store?.logo,
	ogSquareImage: { url: '', width: 56, height: 56 },
	openingHours: ['Monday,Tuesday,Wednesday,Thursday,Friday,Saturday 10:00-20:00'],
	// popularity: data.category?.popularity,
	// postalCode: '764036',
	// price: data.category?.price,
	// priceRange: `${data.category?.price}-${data.category?.mrp}`,
	// ratingCount: 1,
	// ratingValue: +data.category?.ratings + 1,
	// sku: data.category?.sku,
	// streetAddress: 'Padmajyoti Marg, Nandapur Road',
	timeToRead: 0,
	updatedAt: today,
	// weight: { unitCode: '', value: '' },
	// width: { unitCode: '', value: '' },
	// wlwmanifestXmlHref: '',
	metadescription: $page.data.store?.description,
	// article: false,
	canonical: `${$page?.url.href}`,
	datePublished: today,
	description: $page.data.store?.description,
	dnsPrefetch: `//cdn.jsdelivr.net`,
	// entityMeta: null,
	featuredImage: {
		url: $page.data.store?.logo,
		width: 675,
		height: 380,
		caption: $page.data.store?.title
	},
	keywords: $page.data.store?.keywords,
	lastUpdated: today,
	msapplicationTileImage: $page.data.store?.logo,
	ogImage: { url: $page.data.store?.logo, width: 128, height: 56 },
	ogImageSecureUrl: `${$page?.data?.store?.logo}`,
	ogImageType: 'image/jpeg',
	ogSiteName: `${$page.data.origin}/sitemap/sitemap.xml`,
	// productAvailability: `${data.category?.stock}`,
	productBrand: data.searchData,
	productName: data.searchData,
	// productPriceAmount: `${data.category?.price}`,
	productPriceCurrency: `${$page?.data?.store?.currencyCode}`,
	slug: `/`,
	// timeToRead: 0,
	title: data.searchData || 'Buy online in - ' + $page.data.store?.websiteName,
	twitterImage: { url: $page.data.store?.logo }
}

let currentPage = 1
let hidden = true
let reachedLast = false
let selectedFilter
let showFilter = false
let showOnPx = 600
let showSort = false
let y

$: innerWidth = 0

$: if (data?.count === 0) {
	saveSearchData(data?.searchData)
}

async function saveSearchData(searchData) {
	try {
		await PopularSearchService.savePopularSearch({
			id: 'new',
			// popularity: 0,
			text: searchData,
			storeId: $page.data.store?.id,
			origin: $page.data.origin
		})
	} catch (e) {
	} finally {
	}
}

function goTop() {
	// scroll to the top
	window.scroll({ top: 0, behavior: 'smooth' })
}

function scrollContainer() {
	return document.documentElement || document.body
}

function handleOnScroll() {
	if (!scrollContainer()) {
		return
	}

	if (scrollContainer().scrollTop > showOnPx) {
		hidden = false
	} else {
		hidden = true
	}
}

async function sortNow(s) {
	let u = new URL($page.url)

	if (s == 'null' || s == null || s == undefined || s == 'undefined') {
		u.searchParams.delete('sort')
	} else {
		await u.searchParams.set('sort', s)
	}
	// await invalidateAll()
	goto(u.toString())
	window.scroll({ top: 0, behavior: 'smooth' })

	// await goto(`/search?${$page.url.searchParams.toString()}`)
}

async function loadNextPage() {
	if (!reachedLast) {
		let nextPage = currentPage + 1
		$page.url.searchParams.delete('page')
		const searchParams = $page.url.searchParams.toString()

		try {
			data.isLoading = true

			const res = await ProductService.fetchNextPageProducts({
				categorySlug: data.category?.slug,
				origin: $page?.data?.origin,
				storeId: $page?.data?.store?.id,
				nextPage,
				searchParams
			})

			// console.log('res', res)

			const nextPageData = res.nextPageData
			data.products = data?.products?.concat(nextPageData)
			data.count = res?.count
			data.facets = res?.facets
			data.err = !res?.estimatedTotalHits ? 'No result Not Found' : null
			currentPage = currentPage + 1

			if (data.count && data.products?.length === data.count) {
				reachedLast = true
			}
		} catch (e) {
			toast(e, 'error')
		} finally {
			data.isLoading = false
		}
	}
}

async function refreshData() {}

let loadMoreDiv

onMount(() => {
	const observer = new IntersectionObserver((entries) => {
		if (!entries) return
		entries.forEach((entry) => {
			if (
				entry.isIntersecting &&
				data.count &&
				data.products?.length < data.count &&
				!data.isLoading
			) {
				// Do something when the element is intersecting
				loadNextPage()
			}
		})
	})

	if (loadMoreDiv && !$page?.data?.isDesktop) {
		observer.observe(loadMoreDiv)
	}
	// console.log('mmmmmmmmmmmmmmmm')
	// if (!$page?.data?.isDesktop && data.count && data.products?.length < data.count) {
	// 	const intersectionObserver = new IntersectionObserver((entries) => {
	// 		console.log('sssssssssssssssssssss', entries[0].intersectionRatio)
	// 		if (entries[0].intersectionRatio <= 0) return
	// 		// load more content;
	// 		loadNextPage()
	// 	})

	// 	// start observing

	// 	intersectionObserver.observe(document.querySelector('.more'))
	// 	// // @ts-ignore
	// 	// gtag('event', 'view_item', {
	// 	// 	currency: 'USD',
	// 	// 	value: 7.77,
	// 	// 	items: [
	// 	// 		{
	// 	// 			item_id: 'SKU_12345',
	// 	// 			item_name: 'Stan and Friends Tee',
	// 	// 			affiliation: 'Google Merchandise Store',
	// 	// 			coupon: 'SUMMER_FUN',
	// 	// 			currency: 'USD',
	// 	// 			discount: 2.22,
	// 	// 			index: 0,
	// 	// 			item_brand: 'Google',
	// 	// 			item_category: 'Apparel',
	// 	// 			item_category2: 'Adult',
	// 	// 			item_category3: 'Shirts',
	// 	// 			item_category4: 'Crew',
	// 	// 			item_category5: 'Short sleeve',
	// 	// 			item_list_id: 'related_products',
	// 	// 			item_list_name: 'Related Products',
	// 	// 			item_variant: 'green',
	// 	// 			location_id: 'ChIJIQBpAG2ahYAR_6128GcTUEo',
	// 	// 			price: 9.99,
	// 	// 			quantity: 1
	// 	// 		}
	// 	// 	]
	// 	// })

	// 	// // @ts-ignore
	// 	// gtag('event', 'view_item_list', {
	// 	// 	item_list_id: 'related_products',
	// 	// 	item_list_name: 'Related products',
	// 	// 	items: [
	// 	// 		{
	// 	// 			item_id: 'SKU_12345',
	// 	// 			item_name: 'Stan and Friends Tee',
	// 	// 			affiliation: 'Google Merchandise Store',
	// 	// 			coupon: 'SUMMER_FUN',
	// 	// 			currency: 'USD',
	// 	// 			discount: 2.22,
	// 	// 			index: 0,
	// 	// 			item_brand: 'Google',
	// 	// 			item_category: 'Apparel',
	// 	// 			item_category2: 'Adult',
	// 	// 			item_category3: 'Shirts',
	// 	// 			item_category4: 'Crew',
	// 	// 			item_category5: 'Short sleeve',
	// 	// 			item_list_id: 'related_products',
	// 	// 			item_list_name: 'Related Products',
	// 	// 			item_variant: 'green',
	// 	// 			location_id: 'ChIJIQBpAG2ahYAR_6128GcTUEo',
	// 	// 			price: 9.99,
	// 	// 			quantity: 1
	// 	// 		}
	// 	// 	]
	// 	// })
	// }
})

async function goCheckbox(item) {
	if (item === $page.url.searchParams.get('tags')) {
		$page.url.searchParams.set('tags', '')
	} else {
		$page.url.searchParams.set('tags', item)
		$page.url.searchParams.set('page', '1')
	}
	await goto($page.url.toString())
	await invalidateAll()
}

function handleFilterTags() {
	selectedFilter = 'Tags'
	showFilter = true
}
</script>

<SEO {...seoProps} />

<svelte:window bind:scrollY="{y}" bind:innerWidth="{innerWidth}" on:scroll="{handleOnScroll}" />

<CatelogNav me="{$page?.data?.me}" cart="{$page?.data?.cart}" store="{$page?.data?.store}">
	<div class="flex max-w-max flex-col items-start gap-1">
		<h2 class="w-28 truncate font-semibold capitalize leading-4">{data.searchData || ''}</h2>

		<p class="text-xs">
			{#if data.count}
				<b>
					{data.count}
				</b>

				Items
			{:else}
				No Item
			{/if}
		</p>
	</div>
</CatelogNav>

<div class="{showFilter || showSort ? 'h-[93vh] overflow-hidden' : 'h-full min-h-screen'}">
	<!-- Style tags -->

	{#if data.styleTags?.length}
		<div
			class="lg:hidden h-12 flex items-center justify-start px-3 sm:px-10 w-screen overflow-x-auto scrollbar-none fixed top-14 sm:top-20 bg-white z-40 shadow-md">
			<div class="inline-flex gap-2">
				{#each data.styleTags || [] as t}
					{#if t?.key}
						<button
							class="whitespace-nowrap block rounded-full border py-1 px-3 text-xs font-medium uppercase transition duration-300 focus:outline-none
											{$page.url.searchParams.get('tags')?.includes(t?.key)
								? 'bg-primary-500 border-primary-500 text-white'
								: 'bg-white hover:border-primary-500 hover:text-primary-500'}"
							on:click="{() => goCheckbox(t?.key)}">
							{t?.key}
						</button>
					{/if}
				{/each}
			</div>
		</div>
	{/if}

	<!-- Mobile black product count indicator -->

	{#if !hidden && innerWidth <= 1024}
		<button
			transition:fade="{{ duration: 500 }}"
			aria-label="Click to go to top"
			class="fixed top-28 left-[50%] z-40 transform -translate-x-1/2 flex w-28 items-center justify-center gap-1 rounded-full bg-black bg-opacity-60 py-1 px-3 text-xs uppercase text-white transition duration-300 focus:outline-none hover:bg-opacity-80 active:scale-90 sm:top-36 whitespace-nowrap"
			on:click="{goTop}">
			<svg
				xmlns="http://www.w3.org/2000/svg"
				fill="none"
				viewBox="0 0 24 24"
				stroke-width="2"
				stroke="currentColor"
				class="h-3 w-3">
				<path stroke-linecap="round" stroke-linejoin="round" d="M4.5 10.5L12 3m0 0l7.5 7.5M12 3v18"
				></path>
			</svg>

			<span class="flex-1">{data.products?.length} / {data.count}</span>
		</button>
	{/if}

	<div class="mb-10 flex flex-col items-start sm:mb-20 lg:flex-row lg:gap-10 lg:p-10">
		{#if data.facets}
			<DesktopFilter
				facets="{data.facets}"
				priceRange="{data.priceRange}"
				query="{data.query}"
				class="sticky top-24 hidden lg:block"
				on:clearAll="{refreshData}" />

			<MobileFilter
				bind:showFilter="{showFilter}"
				bind:showSort="{showSort}"
				facets="{data.facets}"
				priceRange="{data.priceRange}"
				selected="{selectedFilter}"
				class="fixed bottom-0 border-t z-40 block lg:hidden"
				on:clearAll="{refreshData}" />
		{/if}

		<div
			class="w-full flex-1 sm:px-10 sm:pt-10 lg:pt-0 lg:px-0
			{data.styleTags?.length ? 'mt-12 lg:mt-0' : 'mt-0'}">
			{#if data.products?.length}
				<div class="flex flex-col gap-5">
					<div class="hidden flex-wrap items-center justify-between gap-4 px-3 sm:px-0 lg:flex">
						<!-- Name and count -->

						<h1 class="flex flex-wrap items-center gap-2">
							<span class="text-xl font-bold capitalize md:text-2xl"> {data.searchData} </span>

							-

							<span>
								<span class="font-bold text-2xl">
									{data.count}
								</span>

								Items
							</span>
						</h1>

						<!-- Sort -->

						<div class="flex flex-wrap items-center justify-between">
							<label class="flex items-center gap-2">
								<span>Sort : </span>

								<select
									bind:value="{data.sort}"
									class="max-w-max border-b bg-transparent py-1 pr-2 font-semibold focus:border-primary-500 focus:outline-none hover:border-primary-500"
									on:change="{() => sortNow(data.sort)}">
									{#each sorts as s}
										<option value="{s.val}">{s.name}</option>
									{/each}
								</select>
							</label>
						</div>
					</div>

					<!-- Style tags -->

					{#if data.styleTags?.length}
						<div class="hidden lg:flex flex-wrap items-center gap-2">
							{#each data.styleTags || [] as t}
								{#if t?.key}
									<button
										class="whitespace-nowrap block rounded-full border py-1 px-3 text-xs font-medium uppercase transition duration-300 focus:outline-none
											{$page.url.searchParams.get('tags')?.includes(t?.key)
											? 'bg-primary-500 border-primary-500 text-white'
											: 'bg-white hover:border-primary-500 hover:text-primary-500'}"
										on:click="{() => goCheckbox(t?.key)}">
										{t?.key}
									</button>
								{/if}
							{/each}
						</div>
					{/if}

					<!-- Products -->

					<ul
						class="grid grid-cols-2 items-start border-t sm:flex sm:flex-wrap sm:justify-between sm:gap-3 sm:border-t-0 lg:gap-6">
						{#each data.products as p, ix}
							<li>
								<ProductCard product="{p}" />
							</li>

							<!-- Filter by tags -->

							{#if ix % 40 === 39 && data.facets.all_aggs.tags?.all?.buckets?.length}
								<div
									class="col-span-2 block sm:hidden overflow-x-auto bg-primary-100 scrollbar-none">
									<div class="w-full flex items-center gap-6 p-4">
										<div class="shrink-0">
											<span class="text-lg text-gray-500">Filter by</span>

											<br />

											<span class="text-2xl font-bold">Tags</span>
										</div>

										<ul class="flex max-w-[40rem] shrink-0 flex-wrap gap-2">
											{#each data.facets.all_aggs.tags.all.buckets || [] as t, tx}
												{#if t && tx < 12}
													<button
														type="button"
														class="capitalizefocus:outline-none max-w-max rounded-md bg-white py-2 px-4 text-sm font-semibold"
														on:click="{() => goCheckbox(t.key)}">
														{t.key}
													</button>
												{/if}
											{/each}

											{#if data.facets.all_aggs.tags.all.buckets?.length - 12 > 0}
												<button
													type="button"
													class="font-semibold text-sm text-primary-500 focus:outline-none"
													on:click="{handleFilterTags}">
													+{data.facets.all_aggs.tags.all.buckets?.length - 12} more
												</button>
											{/if}
										</ul>
									</div>
								</div>
							{/if}
						{/each}

						{#each { length: 7 } as _}
							<li class="hidden sm:block">
								<DummyProductCard />
							</li>
						{/each}
					</ul>

					{#if !$page?.data?.isDesktop}
						<!-- <div class="more"> -->
						<div bind:this="{loadMoreDiv}">
							<!-- Dot loading gif -->
							{#if data.isLoading}
								<div class="flex items-center justify-center p-6">
									<img
										src="{dotsLoading}"
										alt="loading"
										class="h-auto w-5 object-contain object-center" />
								</div>
							{/if}
						</div>

						<!-- Reached last -->

						{#if reachedLast}
							<p class="text-gray-500 p-4 text-center">
								<i>~ You have seen all the products ~</i>
							</p>
						{/if}
					{:else}
						<Pagination
							count="{Math.ceil((data?.count || 1) / data.pageSize)}"
							current="{data?.currentPage || 1}"
							providePaddingOnMobile />
					{/if}
				</div>
			{:else}
				<div class="flex items-center justify-center px-3 sm:px-0" style="height: 60vh;">
					<div class="m-10 flex flex-col items-center justify-center text-center">
						<h2 class="mb-10 text-xl capitalize sm:text-2xl lg:text-3xl">
							{#if data.searchData}You searched for "{data.searchData}"{/if}
						</h2>

						<div class="mb-5">
							<img
								src="{noDataAvailable}"
								alt="no data available"
								class="h-60 w-auto object-contain text-xs" />
						</div>

						<h2 class="mb-1 font-semibold">We couldn't find any matches!</h2>

						<p class="mb-5 text-center text-sm text-gray-500">
							Please check the spelling or try searching something else
						</p>

						<PrimaryButton class="text-sm" on:click="{() => goto('/')}">Back to Home</PrimaryButton>
					</div>
				</div>
			{/if}
		</div>
	</div>
</div>
