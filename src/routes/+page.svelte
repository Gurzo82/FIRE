<script lang="ts">
	import { onMount } from 'svelte';
	import { page } from '$app/state';
	import { t } from '$lib/i18n/store.svelte';
	import {
		Heading,
		Breadcrumb,
		BreadcrumbItem,
		Card,
		Button,
		Progressbar,
		Badge
	} from 'flowbite-svelte';
	import {
		HomeSolid,
		ChartMixedDollarSolid,
		UserSolid,
		FireSolid,
		WalletSolid,
		CalendarMonthSolid,
		ArrowRightOutline,
		ChartPieSolid,
		ClockSolid
	} from 'flowbite-svelte-icons';
	import type { Profile } from '$lib/db/index';
	import { getAllProfiles } from '$lib/db/profiles';
	import {
		calculateFireNumber,
		calculateYearsToFire,
		calculateSavingsRate,
		calculateNetWorth,
		projectPortfolio,
		type ProjectionParams,
		type YearlyProjection
	} from '$lib/engine/fire-calculator';
	import { blendedCapitalGainsRate } from '$lib/engine/tax-italy';
	import { getPreset } from '$lib/engine/assumptions';
	import { formatCurrency, formatPercent, formatCompact, formatDate } from '$lib/utils/format';
	import StatCard from '$lib/components/shared/StatCard.svelte';
	import EChart from '$lib/components/simulazione/EChart.svelte';
	import MarketTicker from '$lib/components/dashboard/MarketTicker.svelte';
	import RebalanceAlert from '$lib/components/dashboard/RebalanceAlert.svelte';
	import BenchmarkComparison from '$lib/components/dashboard/BenchmarkComparison.svelte';

	let profile = $state<Profile | undefined>(undefined);
	let loading = $state(true);
	let projections = $state<YearlyProjection[]>([]);

	function getNetWorth(p: Profile): number {
		const port = p.portfolio as unknown as Record<string, number>;
		return calculateNetWorth(port);
	}

	// Computed values
	let netWorth = $derived(profile ? getNetWorth(profile) : 0);
	let fireNumber = $derived(
		profile ? calculateFireNumber(profile.fireExpenses, profile.simulation.withdrawalRate) : 0
	);
	let fireProgress = $derived(fireNumber > 0 ? Math.min(100, (netWorth / fireNumber) * 100) : 0);
	let annualSavings = $derived(
		profile ? profile.annualIncome + profile.otherIncome - profile.annualExpenses : 0
	);
	let yearsToFire = $derived(
		profile
			? calculateYearsToFire(
					netWorth,
					annualSavings,
					profile.simulation.expectedReturn,
					fireNumber
				)
			: -1
	);
	let savingsRate = $derived(
		profile ? calculateSavingsRate(profile.annualIncome + profile.otherIncome, profile.annualExpenses) : 0
	);
	let monthlyFireIncome = $derived(
		fireNumber > 0 && profile
			? (fireNumber * profile.simulation.withdrawalRate) / 12
			: 0
	);
	let currentAge = $derived(profile ? new Date().getFullYear() - profile.birthYear : 0);

	// Chart options
	let projectionChartOptions = $derived.by(() => {
		if (projections.length === 0) return {};
		const retirementYear = profile
			? new Date().getFullYear() + (profile.retirementAge - currentAge)
			: 0;
		return {
			textStyle: {
				fontFamily: 'Inter, ui-sans-serif, system-ui, -apple-system, sans-serif'
			},
			tooltip: {
				trigger: 'axis',
				backgroundColor: 'rgba(17, 24, 39, 0.92)',
				borderWidth: 0,
				textStyle: { color: '#f9fafb', fontSize: 12 },
				formatter: (params: any) => {
					const p = params[0];
					return `<strong>${p.name}</strong><br/>Portafoglio: ${formatCurrency(p.value)}`;
				}
			},
			grid: { left: '3%', right: '4%', bottom: '3%', containLabel: true },
			xAxis: {
				type: 'category',
				data: projections.map((p) => p.year.toString()),
				axisLabel: { interval: 'auto' }
			},
			yAxis: {
				type: 'value',
				axisLabel: {
					formatter: (v: number) => formatCompact(v)
				}
			},
			series: [
				{
					name: 'Portafoglio',
					type: 'line',
					data: projections.map((p) => Math.round(p.portfolio)),
					smooth: true,
					areaStyle: {
						opacity: 0.15
					},
					lineStyle: { width: 2.5 },
					itemStyle: { color: '#3b82f6' },
					markLine: retirementYear
						? {
								silent: true,
								data: [{ xAxis: retirementYear.toString(), label: { formatter: 'FIRE' } }],
								lineStyle: { color: '#ef4444', type: 'dashed' }
							}
						: undefined
				}
			]
		};
	});

	let allocationChartOptions = $derived.by(() => {
		if (!profile) return {};
		const labels: Record<string, string> = {
			stocks: 'Azioni',
			bonds: 'Obbligazioni',
			cash: 'Liquidita\'',
			realEstate: 'Immobiliare',
			gold: 'Oro',
			crypto: 'Crypto',
			pensionFund: 'Fondo Pensione',
			tfr: 'TFR',
			other: 'Altro'
		};
		const colors = [
			'#3b82f6', '#10b981', '#f59e0b', '#8b5cf6',
			'#f97316', '#06b6d4', '#ec4899', '#6366f1', '#94a3b8'
		];
		const data = Object.entries(profile.portfolio)
			.filter(([, v]) => v > 0)
			.map(([k, v]) => ({ name: labels[k] || k, value: v }));

		return {
			textStyle: {
				fontFamily: 'Inter, ui-sans-serif, system-ui, -apple-system, sans-serif'
			},
			tooltip: {
				trigger: 'item',
				backgroundColor: 'rgba(17, 24, 39, 0.92)',
				borderWidth: 0,
				textStyle: { color: '#f9fafb', fontSize: 12 },
				formatter: (p: any) => `${p.name}: ${formatCurrency(p.value)} (${p.percent.toFixed(1)}%)`
			},
			legend: {
				orient: 'vertical',
				right: '5%',
				top: 'center',
				textStyle: { fontSize: 12 }
			},
			series: [
				{
					type: 'pie',
					radius: ['40%', '70%'],
					center: ['35%', '50%'],
					avoidLabelOverlap: true,
					itemStyle: {
						borderRadius: 6,
						borderColor: '#fff',
						borderWidth: 2
					},
					label: { show: false },
					emphasis: {
						label: { show: true, fontSize: 14, fontWeight: 'bold' }
					},
					data,
					color: colors
				}
			]
		};
	});

	onMount(async () => {
		try {
			const profiles = await getAllProfiles();
			if (profiles.length > 0) {
				profile = profiles[0];
				// Generate projections
				const totalContributions = Object.values(profile.monthlyContributions).reduce(
					(s, v) => s + (v || 0),
					0
				);
				const params: ProjectionParams = {
					initialPortfolio: getNetWorth(profile),
					annualContribution: totalContributions * 12,
					annualExpenses: profile.fireExpenses,
					expectedReturn: profile.simulation.expectedReturn,
					inflationRate: profile.simulation.inflationRate,
					taxRate: blendedCapitalGainsRate(
						profile.portfolio as unknown as Record<string, number>,
						getPreset(profile.assumptionsId).capital
					),
					withdrawalRate: profile.simulation.withdrawalRate,
					currentAge: new Date().getFullYear() - profile.birthYear,
					retirementAge: profile.retirementAge,
					lifeExpectancy: profile.lifeExpectancy,
					startYear: new Date().getFullYear()
				};
				projections = projectPortfolio(params);
			}
		} catch (e) {
			console.error('Errore caricamento profilo:', e);
		} finally {
			loading = false;
		}
	});
</script>

<svelte:head>
	<title>Dashboard - FIRE Planner</title>
</svelte:head>

<Breadcrumb class="mb-4">
	<BreadcrumbItem href="/" home>Dashboard</BreadcrumbItem>
</Breadcrumb>

{#if loading}
	<div class="flex items-center justify-center py-20">
		<div class="text-gray-500 dark:text-gray-400 text-lg">Caricamento...</div>
	</div>
{:else if !profile}
	<!-- No profile: Getting Started -->
	<div class="max-w-2xl mx-auto py-12 text-center">
		<div class="mb-6">
			<FireSolid class="w-16 h-16 text-primary-500 mx-auto mb-4" />
			<Heading tag="h1" class="mb-4">Benvenuto nel FIRE Planner</Heading>
			<p class="text-lg text-gray-600 dark:text-gray-400 mb-8">
				Per iniziare, crea il tuo profilo finanziario. Inserisci le tue entrate, spese e patrimonio
				per calcolare il tuo percorso verso l'indipendenza finanziaria.
			</p>
		</div>

		<div class="grid grid-cols-1 md:grid-cols-3 gap-4 mb-8">
			<Card>
				<div class="text-center">
					<div class="rounded-full bg-primary-100 dark:bg-primary-900/30 w-12 h-12 flex items-center justify-center mx-auto mb-3">
						<UserSolid class="w-6 h-6 text-primary-500" />
					</div>
					<h5 class="font-semibold text-gray-900 dark:text-white mb-1">1. Profilo</h5>
					<p class="text-sm text-gray-500 dark:text-gray-400">Configura dati personali e finanziari</p>
				</div>
			</Card>
			<Card>
				<div class="text-center">
					<div class="rounded-full bg-green-100 dark:bg-green-900/30 w-12 h-12 flex items-center justify-center mx-auto mb-3">
						<ChartMixedDollarSolid class="w-6 h-6 text-green-500" />
					</div>
					<h5 class="font-semibold text-gray-900 dark:text-white mb-1">2. Calcola</h5>
					<p class="text-sm text-gray-500 dark:text-gray-400">Scopri il tuo numero FIRE</p>
				</div>
			</Card>
			<Card>
				<div class="text-center">
					<div class="rounded-full bg-red-100 dark:bg-red-900/30 w-12 h-12 flex items-center justify-center mx-auto mb-3">
						<FireSolid class="w-6 h-6 text-red-500" />
					</div>
					<h5 class="font-semibold text-gray-900 dark:text-white mb-1">3. Simula</h5>
					<p class="text-sm text-gray-500 dark:text-gray-400">Testa la robustezza del piano</p>
				</div>
			</Card>
		</div>

		<Button href="/profilo/" size="lg" class="gap-2">
			<UserSolid class="w-5 h-5" />
			Crea il tuo Profilo
			<ArrowRightOutline class="w-4 h-4 ml-1" />
		</Button>
	</div>
{:else}
	<!-- Dashboard con profilo -->

	<!-- Welcome Section -->
	<div class="flex flex-col sm:flex-row sm:items-center sm:justify-between mb-6">
		<div>
			<Heading tag="h1" class="mb-1">Ciao, {profile.name}!</Heading>
			<p class="text-gray-500 dark:text-gray-400">{formatDate(new Date())} &mdash; Il tuo riepilogo FIRE</p>
		</div>
		<Badge color="blue" large class="mt-2 sm:mt-0">
			{currentAge} anni
		</Badge>
	</div>

	<!-- Program Description -->
	<Card class="max-w-none mb-6 p-4 bg-gradient-to-br from-primary-600 via-primary-700 to-primary-900 border-0 text-white shadow-lg">
		<div class="flex items-start gap-3">
			<FireSolid class="w-6 h-6 text-blue-500 mt-0.5 flex-shrink-0" />
			<div class="text-sm text-primary-50 leading-relaxed">
				<p class="font-medium text-white mb-2">
					Cos'è FIRE Planner e come ti aiuta
				</p>
				<p class="mb-2">
					<strong>FIRE Planner</strong> è uno strumento gratuito e privato per pianificare la tua indipendenza finanziaria (Financial Independence, Retire Early) 
					con calcoli specifici per il <strong>contesto fiscale e previdenziale italiano</strong>.
				</p>
				<p class="mb-2">
					Il programma calcola il tuo <strong>FIRE Number</strong> (il capitale necessario per vivere di rendita), 
					il <strong>tasso di risparmio</strong>, gli <strong>anni al FIRE</strong> e proietta l'evoluzione del tuo patrimonio nel tempo.
				</p>
				<p class="mb-2">
					Include simulazioni <strong>Monte Carlo</strong> per testare la robustezza del piano contro i ribassi di mercato,
					ottimizzazione fiscale dei prelievi (IRPEF, capital gains 26%/12,5%, bollo, IVAFE), 
					stima della pensione INPS e calcolo del gap pensionistico.
				</p>
				<p class="text-xs text-primary-100">
					🔒 <strong>Privacy totale:</strong> tutti i calcoli avvengono nel browser, i dati restano sul tuo dispositivo (IndexedDB) e non vengono mai inviati a server esterni.
				</p>
			</div>
		</div>
	</Card>

	<!-- Market Ticker -->
	<MarketTicker />

	<!-- Rebalance Reminder Alert -->
	<RebalanceAlert />

	<!-- Summary Cards Grid -->
	<div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4 mb-8">
		<div class="transition-transform hover:-translate-y-1">
		<StatCard
			title="Patrimonio Netto"
			value={formatCurrency(netWorth)}
			subtitle="Totale investimenti"
			icon={WalletSolid}
			color="blue"
		/>
		</div>
		<div class="transition-transform hover:-translate-y-1">
		<StatCard
			title="FIRE Number"
			value={formatCurrency(fireNumber)}
			subtitle="{formatPercent(profile.simulation.withdrawalRate * 100)} tasso di prelievo"
			icon={FireSolid}
			color="red"
		/>
		</div>
		<div class="transition-transform hover:-translate-y-1">
		<StatCard
			title="Progresso FIRE"
			value="{fireProgress.toFixed(1)}%"
			subtitle="{formatCurrency(fireNumber - netWorth)} rimanenti"
			icon={ChartPieSolid}
			color={fireProgress >= 75 ? 'green' : fireProgress >= 50 ? 'yellow' : 'primary'}
			trend={fireProgress >= 50 ? 'up' : 'neutral'}
		/>
		</div>
		<div class="transition-transform hover:-translate-y-1">
		<StatCard
			title="Anni al FIRE"
			value={yearsToFire === -1 ? 'N/D' : yearsToFire === 0 ? 'Raggiunto!' : `${yearsToFire} anni`}
			subtitle={yearsToFire > 0 ? `Previsto nel ${new Date().getFullYear() + yearsToFire}` : ''}
			icon={CalendarMonthSolid}
			color={yearsToFire === 0 ? 'green' : yearsToFire <= 10 ? 'yellow' : 'primary'}
		/>
		</div>
		<div class="transition-transform hover:-translate-y-1">
		<StatCard
			title="Tasso di Risparmio"
			value={formatPercent(savingsRate * 100)}
			subtitle="{formatCurrency(annualSavings)}/anno risparmiati"
			icon={ChartMixedDollarSolid}
			color={savingsRate >= 0.5 ? 'green' : savingsRate >= 0.3 ? 'yellow' : 'red'}
			trend={savingsRate >= 0.3 ? 'up' : 'down'}
		/>
		</div>
		<div class="transition-transform hover:-translate-y-1">
		<StatCard
			title="Reddito Mensile FIRE"
			value={formatCurrency(monthlyFireIncome)}
			subtitle="Stima in base al SWR"
			icon={HomeSolid}
			color="green"
		/>
		</div>
	</div>

	<!-- FIRE Progress Bar -->
	<Card class="max-w-none mb-8">
		<div class="flex items-center justify-between mb-3">
			<h5 class="text-lg font-semibold text-gray-900 dark:text-white">Progresso verso il FIRE</h5>
			<span class="text-2xl font-bold text-primary-600 dark:text-primary-400">{fireProgress.toFixed(1)}%</span>
		</div>
		<div class="mb-2">
			<Progressbar
				progress={fireProgress}
				size="h-5"
				labelOutside=""
				color={fireProgress >= 75 ? 'green' : fireProgress >= 50 ? 'yellow' : 'blue'}
			/>
		</div>
		<div class="flex justify-between text-sm text-gray-500 dark:text-gray-400">
			<span>Attuale: {formatCurrency(netWorth)}</span>
			<span>Obiettivo: {formatCurrency(fireNumber)}</span>
		</div>
	</Card>

	<!-- Charts Row -->
	<div class="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-8">
		<!-- Portfolio Projection Chart -->
		<Card class="max-w-none">
			<h5 class="text-lg font-semibold text-gray-900 dark:text-white mb-4">
				Proiezione del Portafoglio
			</h5>
			{#if projections.length > 0}
				<EChart options={projectionChartOptions} height="350px" />
				<p class="mt-3 text-xs text-gray-500 dark:text-gray-400 leading-relaxed">
					<strong>Cosa mostra:</strong> l'evoluzione stimata del tuo patrimonio anno per anno, dalla situazione attuale fino all'aspettativa di vita.
					La linea <span class="text-green-600 dark:text-green-400 font-medium">verde</span> rappresenta la fase di <strong>accumulo</strong> (contributi + rendimenti), 
					la linea <span class="text-amber-600 dark:text-amber-400 font-medium">arancione</span> la fase di <strong>decumulo</strong> (prelievi per spese).
					La linea tratteggiata <span class="text-red-600 dark:text-red-400 font-medium">rossa</span> indica il <strong>Target FIRE</strong> aggiustato per l'inflazione.
				</p>
				<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">
					<strong>A cosa serve:</strong> visualizzare se e quando il patrimonio supera il target FIRE, 
					quanto dura in decumulo e l'impatto di tasse/inflazione. Passa il mouse per vedere i dettagli anno per anno.
				</p>
			{:else}
				<p class="text-gray-500 dark:text-gray-400 py-8 text-center">
					Dati insufficienti per la proiezione
				</p>
			{/if}
		</Card>

		<!-- Asset Allocation Pie Chart -->
		<Card class="max-w-none">
			<h5 class="text-lg font-semibold text-gray-900 dark:text-white mb-4">
				Allocazione del Patrimonio
			</h5>
			{#if netWorth > 0}
				<EChart options={allocationChartOptions} height="350px" />
				<p class="mt-3 text-xs text-gray-500 dark:text-gray-400 leading-relaxed">
					<strong>Cosa mostra:</strong> la composizione del tuo portafoglio per classe di asset 
					(Azioni, Obbligazioni, Liquidità, Immobiliare, Oro, Crypto, Fondo Pensione, TFR, Altro).
					Ogni fetta è proporzionale al valore corrente di quell'asset.
				</p>
				<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">
					<strong>A cosa serve:</strong> verificare la diversificazione e il profilo di rischio del portafoglio. 
					Un'eccessiva concentrazione in una singola classe aumenta la volatilità. 
					Clicca sulla legenda per isolare/nascondere una classe.
				</p>
			{:else}
				<p class="text-gray-500 dark:text-gray-400 py-8 text-center">
					Nessun patrimonio inserito
				</p>
			{/if}
		</Card>
	</div>

	<!-- FIRE Benchmark Comparison -->
	<div class="mb-8">
		<BenchmarkComparison
			userSavingsRate={savingsRate}
			userFireNumber={fireNumber}
			userYearsToFire={yearsToFire}
		/>
	</div>

	<!-- Quick Actions -->
	<Heading tag="h2" class="text-xl mb-4">Azioni Rapide</Heading>
	<div class="grid grid-cols-1 md:grid-cols-3 gap-4">
		<Card class="max-w-none">
			<div class="flex items-center gap-3 mb-3">
				<div class="p-2 rounded-lg bg-blue-100 dark:bg-blue-900/30">
					<ChartMixedDollarSolid class="w-5 h-5 text-blue-500" />
				</div>
				<h5 class="text-lg font-semibold text-gray-900 dark:text-white">Nuova Simulazione</h5>
			</div>
			<p class="text-sm text-gray-500 dark:text-gray-400 mb-4">
				Esegui una simulazione Monte Carlo per testare il tuo piano FIRE.
			</p>
			<Button href="/simulazione/" outline size="sm" class="gap-1">
				Vai alla Simulazione
				<ArrowRightOutline class="w-3.5 h-3.5" />
			</Button>
		</Card>

		<Card class="max-w-none">
			<div class="flex items-center gap-3 mb-3">
				<div class="p-2 rounded-lg bg-green-100 dark:bg-green-900/30">
					<UserSolid class="w-5 h-5 text-green-500" />
				</div>
				<h5 class="text-lg font-semibold text-gray-900 dark:text-white">Modifica Profilo</h5>
			</div>
			<p class="text-sm text-gray-500 dark:text-gray-400 mb-4">
				Aggiorna i dati finanziari del tuo profilo per risultati piu' accurati.
			</p>
			<Button href="/profilo/" outline color="green" size="sm" class="gap-1">
				Vai al Profilo
				<ArrowRightOutline class="w-3.5 h-3.5" />
			</Button>
		</Card>

		<Card class="max-w-none">
			<div class="flex items-center gap-3 mb-3">
				<div class="p-2 rounded-lg bg-purple-100 dark:bg-purple-900/30">
					<ClockSolid class="w-5 h-5 text-purple-500" />
				</div>
				<h5 class="text-lg font-semibold text-gray-900 dark:text-white">Guida Rapida</h5>
			</div>
			<p class="text-sm text-gray-500 dark:text-gray-400 mb-4">
				Scopri come utilizzare al meglio il FIRE Planner e le sue funzionalita'.
			</p>
			<Button href="/guida/" outline color="purple" size="sm" class="gap-1">
				Vai alla Guida
				<ArrowRightOutline class="w-3.5 h-3.5" />
			</Button>
		</Card>
	</div>
{/if}
