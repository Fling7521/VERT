<script lang="ts">
	import { onMount } from "svelte";
	import { goto, beforeNavigate, afterNavigate } from "$app/navigation";

	import {
		PUB_PLAUSIBLE_URL,
		PUB_HOSTNAME,
		PUB_DONATION_URL,
	} from "$env/static/public";
	import { VERT_NAME } from "$lib/consts";
	import * as Layout from "$lib/components/layout";
	import * as Navbar from "$lib/components/layout/Navbar";
	import featuredImage from "$lib/assets/VERT_Feature.webp";
	import { Settings } from "$lib/sections/settings/index.svelte";
	import {
		files,
		isMobile,
		effects,
		theme,
		dropping,
		vertdLoaded,
	} from "$lib/store/index.svelte";
	import "$lib/css/app.scss";
	import { browser } from "$app/environment";
	import { page } from "$app/state";

	let { children, data } = $props();
	let enablePlausible = $state(false);

	let scrollPositions = new Map<string, number>();

	beforeNavigate((nav) => {
		if (!nav.from || !$isMobile) return;
		scrollPositions.set(nav.from.url.pathname, window.scrollY);
	});

	afterNavigate((nav) => {
		if (!$isMobile) return;
		const scrollY = nav.to
			? scrollPositions.get(nav.to.url.pathname) || 0
			: 0;
		window.scrollTo(0, scrollY);
	});

	const dropFiles = (e: DragEvent) => {
		e.preventDefault();
		dropping.set(false);
		if (page.url.pathname !== "/jpegify/") {
			const oldLength = files.files.length;
			files.add(e.dataTransfer?.files);
			if (oldLength !== files.files.length) goto("/convert");
		} else {
			files.add(e.dataTransfer?.files);
		}
	};

	const handleDrag = (e: DragEvent, drag: boolean) => {
		e.preventDefault();
		dropping.set(drag);
	};

	onMount(() => {
		isMobile.set(window.innerWidth <= 768);
		window.addEventListener("resize", () => {
			isMobile.set(window.innerWidth <= 768);
		});

		effects.set(localStorage.getItem("effects") !== "false"); // defaults to true if not set
		theme.set(
			(localStorage.getItem("theme") as "light" | "dark") || "light",
		);

		Settings.instance.load();

		fetch(`${Settings.instance.settings.vertdURL}/api/version`).then(
			(res) => {
				if (res.ok) $vertdLoaded = true;
			},
		);
	});

	$effect(() => {
		// Enable plausible if enabled
		enablePlausible =
			!!PUB_PLAUSIBLE_URL && Settings.instance.settings.plausible;
		if (!enablePlausible && browser) {
			// reset pushState on opt-out so that plausible stops firing events on page navigation
			history.pushState = History.prototype.pushState;
		}
	});
</script>

<svelte:head>
	<title>{VERT_NAME}</title>
	<meta name="theme-color" content="#F2ABEE" />
	<meta
		name="title"
		content="{VERT_NAME} — Free, fast, and awesome file convert"
	/>
	<meta
		name="description"
		content="With VERT you can quickly convert any image, video and audio file. No ads, no tracking, open source, and all processing (other than video) is done on your device."
	/>
	<meta property="og:type" content="website" />
	<meta
		property="og:title"
		content="{VERT_NAME} — Free, fast, and awesome file convert"
	/>
	<meta
		property="og:description"
		content="With VERT you can quickly convert any image, video and audio file. No ads, no tracking, open source, and all processing (other than video) is done on your device."
	/>
	<meta property="og:image" content={featuredImage} />
	<meta property="twitter:card" content="summary_large_image" />
	<meta
		property="twitter:title"
		content="{VERT_NAME} — Free, fast, and awesome file convert"
	/>
	<meta
		property="twitter:description"
		content="With VERT you can quickly convert any image, video and audio file. No ads, no tracking, open source, and all processing (other than video) is done on your device."
	/>
	<meta property="twitter:image" content={featuredImage} />
	<link rel="manifest" href="/manifest.json" />
	{#if enablePlausible}
		<script
			defer
			data-domain={PUB_HOSTNAME || "vert.sh"}
			src="{PUB_PLAUSIBLE_URL}/js/script.js"
		></script>
	{/if}
	{#if data.isAprilFools}
		<style>
			* {
				font-family: "Comic Sans MS", "Comic Sans", cursive !important;
			}
		</style>
	{/if}
</svelte:head>

<!-- FIXME: if user resizes between desktop/mobile, highlight of page disappears (only shows on original size) -->
<div
	class="flex flex-col min-h-screen h-full w-full overflow-x-hidden"
	ondrop={dropFiles}
	ondragenter={(e) => handleDrag(e, true)}
	ondragover={(e) => handleDrag(e, true)}
	ondragleave={(e) => handleDrag(e, false)}
	role="region"
>
	<Layout.UploadRegion />

	<div>
		<Layout.MobileLogo />
		<Navbar.Desktop />
	</div>

	<!-- 
		SvelteKit throws the following warning when developing - safe to ignore as we render the children in this component:
		`<slot />` or `{@render ...}` tag missing — inner content will not be rendered
	-->
	<Layout.PageContent {children} />

	<Layout.Toasts />
	<Layout.Dialogs />

	<div>
		<Layout.Footer />
		<Navbar.Mobile />
	</div>
</div>

<!-- Gradients placed here to prevent it overlapping in transitions -->
<Layout.Gradients />
