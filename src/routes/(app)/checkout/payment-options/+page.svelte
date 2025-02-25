<style>
.frosted-black {
	backdrop-filter: blur(12px);
	background-color: hsla(0, 0%, 0%, 0.5);
}

.z-index {
	z-index: 99999;
}
</style>

<script>
import { toast } from '$lib/utils'
import { fireGTagEvent } from '$lib/utils/gTag'
import { goto } from '$app/navigation'
import { onMount } from 'svelte'
import { page } from '$app/stores'
import { post } from '$lib/utils/api'
import { stripePublishableKey } from '$lib/config'
import CheckoutHeader from '$lib/components/CheckoutHeader.svelte'
import Error from '$lib/components/Error.svelte'
import LazyImg from '$lib/components/Image/LazyImg.svelte'
import logo from '$lib/assets/logo.svg'
import Pricesummary from '$lib/components/Pricesummary.svelte'
import PrimaryButton from '$lib/ui/PrimaryButton.svelte'
import SEO from '$lib/components/SEO/index.svelte'
import Stripe from '$lib/Stripe.svelte'
import { CartService, OrdersService, PaymentMethodService } from '$lib/services'

const seoProps = {
	title: 'Select Payment Option',
	metadescription: 'Choose your payment method'
}

export let data

// console.log('zzzzzzzzzzzzzzzzzz', data)

let errorMessage = 'Select a Payment Method',
	disabled = false,
	showPayWithBankTransfer = false,
	bankPayment = {
		type: 'order',
		reference: '',
		remark: '',
		paymentMethodId: '',
		amount: 0
	},
	selectedPaymentMethod = {
		id: '',
		name: '',
		text: '',
		instructions: '',
		qrcode: '',
		img: ''
	},
	paymentDenied = false,
	razorpayReady = false,
	loading = false

$: if (data.paymentMethods?.length === 1 && data.paymentMethods[0]?.type === 'pg') {
	const pm = data.paymentMethods[0]

	submit(pm)
}

onMount(async () => {
	const razorpayScript = document.createElement('script')
	razorpayScript.setAttribute('src', 'https://checkout.razorpay.com/v1/checkout.js')
	document.head.appendChild(razorpayScript)
	razorpayReady = true
	fireGTagEvent('begin_checkout', data.cart)
})

function paymentMethodChanged(pm) {
	selectedPaymentMethod = pm
	errorMessage = null
}

async function submit(pm) {
	// console.log('pm = ', pm)

	if (!pm || pm === undefined) {
		disabled = true
		errorMessage = 'Please select a payment option'
		toast('Please select a payment option', 'error')
		return
	}

	const paymentMethod = pm.value

	if (paymentMethod === 'COD') {
		try {
			loading = true

			const res = await OrdersService.codCheckout({
				address: data.addressId,
				paymentMethod: 'COD',
				prescription: data.prescription?._id,
				storeId: $page.data.store?.id,
				origin: $page.data.origin
			})

			goto(`/payment/success?id=${res?._id}&status=PAYMENT_SUCCESS&provider=COD`)
		} catch (e) {
			toast(e, 'error')
		} finally {
			loading = false
		}
	} else if (paymentMethod === 'Cashfree') {
		try {
			loading = true
			const res = await OrdersService.cashfreeCheckout({
				address: data.addressId,
				storeId: $page.data.store?.id,
				origin: $page.data.origin
			})
			if (res?.redirectUrl && res?.redirectUrl !== null) {
				goto(`${res?.redirectUrl}`)
			} else {
				toast('Something went wrong', 'error')
			}
		} catch (e) {
			toast(e?.body, 'error')
		} finally {
			loading = false
		}
	} else if (paymentMethod === 'Razorpay') {
		try {
			loading = true
			const rp = await OrdersService.razorpayCheckout({
				address: data.addressId,
				storeId: $page.data.store?.id,
				origin: $page.data.origin
			})

			const options = {
				key: rp.keyId, // Enter the Key ID generated from the Dashboard
				name: 'Litekart.in',
				description: 'Payment for Litekart',
				image: logo,
				amount: rp.amount,
				order_id: rp.id,
				async handler(response) {
					// console.log('response = ', response)

					try {
						const capture = await OrdersService.razorpayCapture({
							rpPaymentId: response.razorpay_payment_id,
							rpOrderId: response.razorpay_order_id,
							storeId: $page.data.store?.id,
							origin: $page.data.origin
						})
						toast('Payment success', 'success')
						goto(`/payment/success?id=${capture._id}`)
					} catch (e) {
						goto(`/payment/failure?ref=/checkout/payment-options?address=${data.addressId}`)
					} finally {
					}
				},
				prefill: {
					name: `${data.me.firstName} ${data.me.lastName}`,
					phone: data.me.phone,
					email: data.me.email || 'help@litekart.in',
					contact: data.me.phone
				},
				notes: {
					address: 'Padmajyoti Marg, Semiliguda, Odisha 764036'
				},
				theme: {
					color: '#112D4E'
				}
			}

			const rzp1 = new Razorpay(options)
			rzp1.open()
		} catch (e) {
			toast(e?.message, 'error')
		} finally {
			loading = false
		}
	} else {
		paymentDenied = true

		setTimeout(() => {
			paymentDenied = false
		}, 820)
	}
}

function checkIfStripeCardValid({ detail }) {
	disabled = !detail
}
</script>

<SEO {...seoProps} />

<Error err="{data.err}" />

<div class="container mx-auto min-h-screen w-full max-w-6xl p-3 py-5 sm:p-10">
	<CheckoutHeader selected="payment" />

	<div class="mt-10 flex flex-col gap-10 md:flex-row md:justify-center xl:gap-20">
		<div class="w-full flex-1">
			<h2 class="mb-5 text-xl font-bold capitalize tracking-wide sm:text-2xl">Payment Options</h2>

			{#if data.paymentMethods?.length}
				<div class="flex w-full flex-col gap-4" class:wiggle="{paymentDenied}">
					{#each data.paymentMethods as pm}
						<label
							class="flex w-full cursor-pointer items-center gap-2 rounded-md border border-gray-300 p-4 shadow-md transition duration-300 hover:bg-primary-50 sm:gap-4">
							<input
								bind:group="{selectedPaymentMethod}"
								type="radio"
								value="{pm}"
								name="group"
								class="h-4 w-4 focus:outline-none focus:ring-0 focus:ring-offset-0"
								on:click="{() => paymentMethodChanged(pm)}" />

							<div class="flex w-full flex-1 items-center justify-between gap-4">
								<div class="flex-1">
									<h2 class="text-xl font-semibold" style="color:{pm.color}">
										{pm.name}
									</h2>

									<p class="mt-1 text-sm text-gray-500">{pm.text}</p>
								</div>

								<div class="shrink-0">
									{#if pm.img}
										<img
											src="{pm.img}"
											alt="{pm.name}"
											width="56"
											height="56"
											class="h-14 w-14 rounded-full border object-cover object-center text-xs" />
									{:else}
										<div
											class="flex h-14 w-14 items-center justify-center rounded-full border bg-gray-200 text-center text-xs uppercase">
											{pm.name}
										</div>
									{/if}
								</div>
							</div>
						</label>

						{#if pm.value === 'stripe'}
							<Stripe
								address="{data.addressId}"
								isStripeSelected="{selectedPaymentMethod.name === 'Stripe'}"
								stripePublishableKey="{stripePublishableKey}"
								on:isStripeCardValid="{checkIfStripeCardValid}" />
						{/if}
					{/each}
				</div>
			{:else}
				<div class="flex flex-col h-1/2 items-center justify-center p-4 text-gray-500 text-center">
					<svg
						xmlns="http://www.w3.org/2000/svg"
						class="mb-2 h-10 w-10"
						fill="none"
						viewBox="0 0 24 24"
						stroke="currentColor"
						stroke-width="2">
						<path
							stroke-linecap="round"
							stroke-linejoin="round"
							d="M9.172 16.172a4 4 0 015.656 0M9 10h.01M15 10h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"
						></path>
					</svg>

					<h6 class="mb-2 font-bold capitalize">We are very sorry!!</h6>

					<p class="text-sm">Payment method is not setup yet, Please contact the store admin</p>
				</div>
			{/if}
		</div>

		<div class="w-full md:w-80 md:shrink-0 md:grow-0">
			<h2 class="text-xl font-bold capitalize tracking-wide sm:text-2xl">Cart Summary</h2>
			{#if data.address}
				<div class="mt-5 border-t pt-5">
					<h5 class="mb-2 text-xl font-bold capitalize tracking-wide">Delivery Address</h5>

					<div class="text-sm font-light">
						<div class="my-1 flex flex-row">
							<h5 class="mr-2 w-20 shrink-0 font-semibold tracking-wide">Name</h5>

							<p>
								{data.address.firstName}
								{data.address.lastName}
							</p>
						</div>

						<div class="flex flex-row">
							<h5 class="mr-2 w-20 shrink-0 font-semibold tracking-wide">Address</h5>

							<p class="flex flex-wrap items-center">
								{#if data.address.address}
									{data.address.address}
								{/if}

								{#if data.address.locality}
									, {data.address.locality}
								{/if}

								{#if data.address.city}
									, {data.address.city}
								{/if}

								{#if data.address.state}
									, {data.address.state}
								{/if}

								{#if data.address.country}
									, {data.address.country}
								{/if}
							</p>
						</div>

						<div class="my-1 flex flex-row">
							<h5 class="mr-2 w-20 shrink-0 font-semibold tracking-wide">Pin</h5>

							<h6>{data.address.zip}</h6>
						</div>

						<div class="my-1 flex flex-row">
							<h5 class="mr-2 w-20 shrink-0 font-semibold tracking-wide">Phone</h5>

							<h6>{data.address.phone}</h6>
						</div>

						<div class="my-1 flex flex-row flex-wrap">
							<h5 class="mr-2 w-20 shrink-0 font-semibold tracking-wide">Email</h5>

							<h6>{data.address.email}</h6>
						</div>
					</div>
				</div>
			{/if}

			{#if data.prescription}
				<div class="mt-5 border-t pt-5">
					<h5 class="mb-2 text-xl font-bold capitalize tracking-wide">Prescription</h5>

					<div class="text-sm font-light">
						{#if data.prescription.name}
							<div class="my-1 flex flex-row">
								<h5 class="mr-2 w-20 shrink-0 font-semibold tracking-wide">Name</h5>

								<p class="flex-1">
									{data.prescription.name}
								</p>
							</div>
						{/if}

						{#if data.prescription.description}
							<div class="my-1 flex flex-row">
								<h5 class="mr-2 w-20 shrink-0 font-semibold tracking-wide">Note</h5>

								<p class="flex-1">
									{data.prescription.description}
								</p>
							</div>
						{/if}

						{#if data.prescription.url}
							<div>
								<LazyImg
									src="{data.prescription.url}"
									alt=""
									height="80"
									class="h-20 w-auto object-contain object-top text-xs" />
							</div>
						{/if}
					</div>
				</div>
			{/if}

			<Pricesummary
				cart="{data.cart}"
				text="{errorMessage || 'Confirm Order'}"
				loading="{loading}"
				hideCheckoutButton="{selectedPaymentMethod.name === 'Stripe'}"
				disabled="{!razorpayReady ||
					!selectedPaymentMethod?.name ||
					(selectedPaymentMethod?.name === 'Stripe' && disabled)}"
				on:submit="{() => submit(selectedPaymentMethod)}" />
		</div>
	</div>
</div>

{#if loading}
	<div
		class="fixed inset-0 z-[100] flex h-screen w-screen items-center justify-center gap-5 bg-black bg-opacity-90 p-5 text-center text-white sm:p-10">
		Please wait... your payment is currently being processed
	</div>
{/if}
