<style>
.mega-menu {
	visibility: hidden;
	transition: 0.3s 0.1s; /* delay of 1 seconds on hover off */
	opacity: 0;
	left: 0;
	position: absolute;
	text-align: left;
	left: 10%;
	right: 20%;
	width: 70%;
	z-index: 9999;
}

/* #hoverable Class Styles
  –––––––––––––––––––––––––––––––––––––––––––––––––– */
.hoverable {
	position: static;
}

.hoverable > a::after {
	padding-left: 6px;
	position: relative;
}

.hoverable:hover .mega-menu {
	visibility: visible;
	opacity: 1;
	transition-delay: 0.1s;
}
</style>

<script>
import { onMount } from 'svelte'
import { toast } from '$lib/utils'
import { page } from '$app/stores'
import { browser } from '$app/environment'
// import { CategoryService } from '$lib/services'
import { provider } from '$lib/config'
import { CategoryService } from '$lib/services'

let megamenu = []
let selectedCategory = ''

onMount(() => {
	getMegaMenu()
})

async function getMegaMenu() {
	if (browser && $page.data.isDesktop) {
		try {
			const localMegamenu = localStorage.getItem('megamenu')

			if (!!localMegamenu && localMegamenu !== 'undefined') {
				megamenu = JSON.parse(localMegamenu)
			} else {
				megamenu = await CategoryService.fetchMegamenuData({
					storeId: $page.data.store?.id,
					origin: $page.data.origin
				})
			}
		} catch (e) {
			toast(e, 'error')
		} finally {
		}
	} else {
		megamenu = []
	}
}
</script>

<ul class="flex flex-row items-center justify-center font-semibold tracking-wide">
	{#each megamenu as category, index}
		<li
			class="hoverable mx-1"
			on:mouseenter="{() => (selectedCategory = category.name)}"
			on:mouseleave="{() => (selectedCategory = '')}">
			<div
				class="itmes-center relative flex h-20 shrink-0 justify-center whitespace-nowrap border-b-4 border-transparent p-2 font-medium uppercase
                    {index % 6 == 0 ? 'hover:border-yellow-500' : ''}
                    {index % 6 == 1 ? 'hover:border-purple-500' : ''}
                    {index % 6 == 2 ? 'hover:border-red-500' : ''}
                    {index % 6 == 3 ? 'hover:border-green-500' : ''}
                    {index % 6 == 4 ? 'hover:border-pink-500' : ''}
                    {index % 6 == 5 ? 'hover:border-blue-500' : ''}
                    {index % 6 == 0 && selectedCategory === category.name
					? 'border-yellow-500'
					: ''}
                    {index % 6 == 1 && selectedCategory === category.name
					? 'border-purple-500'
					: ''}
                    {index % 6 == 2 && selectedCategory === category.name ? 'border-red-500' : ''}
                    {index % 6 == 3 && selectedCategory === category.name ? 'border-green-500' : ''}
                    {index % 6 == 4 && selectedCategory === category.name ? 'border-pink-500' : ''}
                    {index % 6 == 5 && selectedCategory === category.name ? 'border-blue-500' : ''}
                    ">
				<a
					href="/{category.slug}"
					aria-label="Click to route into category related products page"
					data-sveltekit-reload
					class="flex w-full items-center gap-1">
					<!-- Root category -->

					<span>{category.name}</span>

					<!-- Down icon -->

					{#if category.children?.length}
						<svg
							xmlns="http://www.w3.org/2000/svg"
							viewBox="0 0 20 20"
							fill="currentColor"
							class="h-4 w-4 shrink-0 transition duration-300
                            {selectedCategory === category.name ? 'transform -rotate-180' : ''}">
							<path
								fill-rule="evenodd"
								d="M5.23 7.21a.75.75 0 011.06.02L10 11.168l3.71-3.938a.75.75 0 111.08 1.04l-4.25 4.5a.75.75 0 01-1.08 0l-4.25-4.5a.75.75 0 01.02-1.06z"
								clip-rule="evenodd"></path>
						</svg>
					{/if}
				</a>
			</div>

			{#if category.children?.length}
				<div class="mega-menu relative overflow-hidden border-b bg-white shadow-2xl">
					<div class="absolute inset-0 z-0 grid w-full grid-cols-4">
						{#each { length: 4 } as _, ix}
							<div class="{ix % 2 === 0 ? 'bg-white' : 'bg-gray-50'}"></div>
						{/each}
					</div>

					<ul
						class="relative z-10 flex max-h-[75vh] min-h-[50vh] flex-col flex-wrap items-start shadow-inner">
						<!-- 2nd level child category  -->

						{#each category.children as c}
							<li class="mb-2 w-1/4 flex-1 shrink-0 grow-0 p-6 pr-2 text-sm">
								<a
									href="/{c.slug}"
									aria-label="Click to route into category related products page"
									data-sveltekit-reload
									class="mb-2 block w-full
									{index % 6 == 0 ? 'text-yellow-500 ' : ''}
									{index % 6 == 1 ? 'text-purple-500 ' : ''}
									{index % 6 == 2 ? 'text-red-500 ' : ''}
                                    {index % 6 == 3 ? 'text-green-500 ' : ''}
                                    {index % 6 == 4 ? 'text-pink-500 ' : ''}
                                    {index % 6 == 5 ? 'text-blue-500 ' : ''}">
									{c.name}
								</a>

								{#if c && c.children}
									<ul class="flex flex-col flex-wrap items-start gap-0.5">
										<!-- 3rd level child category  -->

										{#each c.children as c1, ixx}
											<li class="w-full">
												<a
													href="/{c1.slug}"
													aria-label="Click to route into category related products page"
													data-sveltekit-reload
													class="block w-full font-light hover:font-medium">
													{c1.name}
												</a>
											</li>
										{/each}
									</ul>
								{/if}
							</li>
						{/each}

						<!-- This dummy divs are required for proper alignment of child elements -->

						{#each { length: 10 } as _}
							<li>
								<div class="h-96 w-1/4 flex-1 shrink-0 grow-0 p-6"></div>
							</li>
						{/each}
					</ul>
				</div>
			{/if}
		</li>
	{/each}
</ul>
