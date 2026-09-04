<script lang="ts">
	import '../app.css';
	import { page } from '$app/state';
	import { onMount } from 'svelte';
	import {
		Sidebar,
		SidebarWrapper,
		SidebarGroup,
		SidebarItem,
		SidebarButton,
		Navbar,
		NavBrand,
		DarkMode,
		Dropdown
	} from 'flowbite-svelte';
	import {
		HomeSolid,
		UserSolid,
		ChartMixedDollarSolid,
		ChartPieSolid,
		FolderSolid,
		ShieldCheckSolid,
		FileChartBarSolid,
		BookOpenSolid,
		CogSolid,
		FireSolid,
		ChartLineUpOutline,
		UsersGroupSolid,
		BuildingSolid,
		LandmarkOutline,
		ScaleBalancedOutline,
		UsersOutline,
		MessagesSolid,
		FileLinesSolid
	} from 'flowbite-svelte-icons';
	import favicon from '$lib/assets/favicon.svg';
	import { t, setLocale, getLocale } from '$lib/i18n/store.svelte';
import { locales } from '$lib/i18n/index';
	import NotificationBanner from '$lib/components/shared/NotificationBanner.svelte';
	import { startReminderChecks, stopReminderChecks, getNotificationPermission } from '$lib/utils/notifications';

	let { children } = $props();

	let sidebarOpen = $state(false);
	let langMenuOpen = $state(false);

	onMount(() => {
		if (getNotificationPermission() === 'granted') {
			startReminderChecks();
		}
		return () => stopReminderChecks();
	});

	const iconClass = 'w-5 h-5 text-gray-500 dark:text-gray-400';

	function isActive(itemHref: string): boolean {
		const path = page.url.pathname;
		if (itemHref === '/') return path === '/';
		return path.startsWith(itemHref);
	}

	// Titolo per-pagina (browser tab + crawler che eseguono JS). I default OG e
	// la description statica vivono in app.html (per unfurler/no-JS, dato che
	// ssr=false). Qui niente OG per non duplicare i tag a runtime.
	const SITE_NAME = 'FIRE Planner';
	const seoByPath: Record<string, { title: string; description: string }> = {
		'/': { title: "FIRE Planner — Indipendenza finanziaria con fiscalita' italiana", description: "Pianificatore FIRE gratuito e privato per il contesto fiscale e previdenziale italiano." },
		'/profilo': { title: 'Profilo finanziario — FIRE Planner', description: 'Inserisci reddito, spese, patrimonio e pensione per calcolare il tuo percorso FIRE.' },
		'/calcolatore': { title: 'Calcolatore FIRE — FIRE Planner', description: 'FIRE number, Coast FIRE, anni al FIRE e strategie di prelievo (4%, VPW, Guyton-Klinger, CAPE).' },
		'/simulazione': { title: 'Simulazione Monte Carlo — FIRE Planner', description: 'Migliaia di simulazioni con dati storici reali e tasso di successo del piano FIRE.' },
		'/scenari': { title: 'Scenari — FIRE Planner', description: 'Crea e confronta scenari FIRE con parametri diversi.' },
		'/rischi': { title: 'Scenari di rischio (stress test) — FIRE Planner', description: "Metti alla prova il piano: crash, inflazione, sequenza rendimenti, longevita'." },
		'/dati-storici': { title: 'Dati storici dei mercati — FIRE Planner', description: 'Rendimenti, correlazioni e CAPE dal 1928 a oggi.' },
		'/pensione': { title: 'Pensione INPS — FIRE Planner', description: 'Stima della pensione contributiva e del gap pensionistico.' },
		'/fondo-pensione': { title: 'Fondo pensione e TFR — FIRE Planner', description: "Deducibilita', tassazione agevolata e rendita del fondo pensione." },
		'/contenitori': { title: 'Confronto contenitori fiscali — FIRE Planner', description: 'Conto titoli, fondo pensione, BTP: quale conviene per il decumulo.' },
		'/confronto-profili': { title: 'Confronto profili — FIRE Planner', description: 'Confronta piani e profili FIRE affiancati.' },
		'/performance': { title: 'Performance del portafoglio — FIRE Planner', description: 'Rendimento time-weighted e confronto col benchmark.' },
		'/guida': { title: 'Guida al FIRE in Italia — FIRE Planner', description: "Guida completa al FIRE con focus su fiscalita' e previdenza italiana." },
		'/impostazioni': { title: 'Impostazioni — FIRE Planner', description: 'Import/export, lingua, valuta, assistente AI e backup.' },
		'/community': { title: 'Benchmark e condivisione — FIRE Planner', description: 'Confronta il tuo piano tramite scenari condivisi via link.' }
	};
	const seo = $derived(seoByPath[page.url.pathname] ?? { title: SITE_NAME, description: '' });
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
	<title>{seo.title}</title>
</svelte:head>

<div class="flex flex-col h-screen">
	<!-- Top Navbar -->
	<Navbar class="border-b border-gray-200 dark:border-gray-700 z-30">
		<div class="flex items-center gap-2">
			<SidebarButton onclick={() => (sidebarOpen = !sidebarOpen)} />
			<NavBrand href="/">
				<FireSolid class="w-6 h-6 text-primary-600 dark:text-primary-400 me-2" />
				<span class="self-center whitespace-nowrap text-xl font-semibold dark:text-white">
					FIRE Planner
				</span>
			</NavBrand>
		</div>
		<div class="flex items-center gap-2">
			<!-- Language Switcher -->
			<button
				id="language-switcher-button"
				type="button"
				aria-expanded={langMenuOpen}
				aria-haspopup="true"
				aria-label={t('settings.language')}
				class="inline-flex items-center gap-1 px-3 py-1.5 text-sm font-medium text-gray-700 dark:text-gray-200 bg-white dark:bg-gray-800 border border-gray-300 dark:border-gray-600 rounded-lg hover:bg-gray-50 dark:hover:bg-gray-700 focus:outline-none focus:ring-2 focus:ring-primary-500"
			>
				<span class="text-base">{locales.find(l => l.code === getLocale())?.flag || '🇮🇹'}</span>
				<span class="hidden sm:inline">{locales.find(l => l.code === getLocale())?.label || 'Italiano'}</span>
			</button>
			<Dropdown
				triggeredBy="#language-switcher-button"
				placement="bottom-end"
				bind:isOpen={langMenuOpen}
			>
				<div class="min-w-[120px]">
					<ul class="py-1 text-sm text-gray-700 dark:text-gray-200" role="menu" aria-orientation="vertical" aria-label={t('settings.chooseLanguage')}>
						{#each locales as locale}
							<li>
								<button
									type="button"
									class="flex w-full items-center gap-2 px-4 py-2 hover:bg-gray-100 dark:hover:bg-gray-700 {getLocale() === locale.code ? 'bg-primary-50 dark:bg-primary-900/30 text-primary-700 dark:text-primary-300' : ''}"
									role="menuitem"
									onclick={() => { setLocale(locale.code); langMenuOpen = false; }}
								>
									<span class="text-base">{locale.flag}</span>
									<span>{locale.label}</span>
									{#if getLocale() === locale.code}
										<svg class="ml-auto w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24" aria-hidden="true">
											<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7" />
										</svg>
									{/if}
								</button>
							</li>
						{/each}
					</ul>
				</div>
			</Dropdown>
			<DarkMode ariaLabel="Cambia tema chiaro/scuro" />
		</div>
	</Navbar>

	<!-- Sidebar + Content -->
	<div class="flex flex-1 overflow-hidden">
		<Sidebar
			isOpen={sidebarOpen}
			closeSidebar={() => (sidebarOpen = false)}
			activeUrl={page.url.pathname}
			class="z-20 top-16"
			divClass="max-h-[calc(100dvh-4rem)]"
		>
			<SidebarWrapper class="py-2">
				<SidebarGroup>
					<SidebarItem label={t('nav.dashboard')} href="/" active={isActive('/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<HomeSolid class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.profile')} href="/profilo/" active={isActive('/profilo/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<UserSolid class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.calculator')} href="/calcolatore/" active={isActive('/calcolatore/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<ChartMixedDollarSolid class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.simulation')} href="/simulazione/" active={isActive('/simulazione/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<ChartPieSolid class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.scenarios')} href="/scenari/" active={isActive('/scenari/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<FolderSolid class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.risks')} href="/rischi/" active={isActive('/rischi/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<ShieldCheckSolid class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.performance')} href="/performance/" active={isActive('/performance/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<ChartLineUpOutline class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.history')} href="/dati-storici/" active={isActive('/dati-storici/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<FileChartBarSolid class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.pensionFund')} href="/fondo-pensione/" active={isActive('/fondo-pensione/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<BuildingSolid class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.inpsPension')} href="/pensione/" active={isActive('/pensione/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<LandmarkOutline class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.containers')} href="/contenitori/" active={isActive('/contenitori/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<ScaleBalancedOutline class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.profileCompare')} href="/confronto-profili/" active={isActive('/confronto-profili/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<UsersOutline class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.community')} href="/community/" active={isActive('/community/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<UsersGroupSolid class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.guide')} href="/guida/" active={isActive('/guida/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<BookOpenSolid class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.aiAssistant')} href="/assistente/" active={isActive('/assistente/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<MessagesSolid class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.methodology')} href="/metodologia/" active={isActive('/metodologia/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<FileLinesSolid class={iconClass} />{/snippet}
					</SidebarItem>
					<SidebarItem label={t('nav.settings')} href="/impostazioni/" active={isActive('/impostazioni/')} onclick={() => (sidebarOpen = false)}>
						{#snippet icon()}<CogSolid class={iconClass} />{/snippet}
					</SidebarItem>
				</SidebarGroup>
			</SidebarWrapper>
		</Sidebar>

		<!-- Main Content Area -->
		<main class="flex-1 overflow-y-auto p-4 md:p-6 md:ml-64">
			<NotificationBanner />
			<!-- Error boundary: un errore in una sezione non deve azzerare l'intera app.
			     I dati restano in IndexedDB (sul dispositivo), quindi e' sicuro riprovare. -->
			<svelte:boundary>
				{@render children()}
				{#snippet failed(error, reset)}
					<div
						class="max-w-2xl mx-auto mt-8 rounded-lg border border-red-300 bg-red-50 p-6 text-center dark:border-red-800 dark:bg-red-950"
					>
						<h2 class="mb-2 text-lg font-semibold text-red-800 dark:text-red-200">
							Si è verificato un errore in questa sezione
						</h2>
						<p class="mb-4 text-sm text-red-700 dark:text-red-300">
							I tuoi dati sono al sicuro (restano nel browser). Puoi riprovare oppure ricaricare la
							pagina.
						</p>
						<p class="mb-4 break-words text-xs text-gray-500 dark:text-gray-400">
							{error instanceof Error ? error.message : String(error)}
						</p>
						<button
							class="rounded-md bg-red-600 px-4 py-2 text-sm font-medium text-white hover:bg-red-700"
							onclick={reset}
						>
							Riprova
						</button>
					</div>
				{/snippet}
			</svelte:boundary>
		</main>
	</div>
</div>
