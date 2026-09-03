<script lang="ts">
	import { onMount } from 'svelte';
	import {
		Heading,
		Breadcrumb,
		BreadcrumbItem,
		Alert,
		Spinner,
		Tabs,
		TabItem,
		Toggle,
		Helper,
		Label,
		Input
	} from 'flowbite-svelte';
	import {
		InfoCircleSolid,
		ChartPieOutline,
		ChartMixedDollarSolid,
		TableColumnSolid,
		CashSolid
	} from 'flowbite-svelte-icons';
	import { formatCurrency } from '$lib/utils/format';

	import type { Profile } from '$lib/db/index';
	import { getAllProfiles } from '$lib/db/profiles';
	import {
		calculateFireNumber,
		calculateFireNumberWithPension,
		calculateYearsToFire,
		calculateSavingsRate,
		calculateNetWorth,
		calculateLiquidNetWorth,
		calculateIlliquidNetWorth,
		projectPortfolio,
		calculateCoastFireNumber,
		type YearlyProjection
	} from '$lib/engine/fire-calculator';
	import { optimizeWithdrawalOrder, blendedCapitalGainsRate, type WithdrawalPlan } from '$lib/engine/tax-italy';
	import { DEFAULT_2026, getPreset, type AssumptionSet } from '$lib/engine/assumptions';

	import FireHero from '$lib/components/calcolatore/FireHero.svelte';
	import AssumptionsPanel from '$lib/components/shared/AssumptionsPanel.svelte';
	import SensitivityTornado from '$lib/components/calcolatore/SensitivityTornado.svelte';
	import MetricsGrid from '$lib/components/calcolatore/MetricsGrid.svelte';
	import FireVariants from '$lib/components/calcolatore/FireVariants.svelte';
	import WithdrawalStrategySelector from '$lib/components/calcolatore/WithdrawalStrategySelector.svelte';
	import ParameterControls from '$lib/components/calcolatore/ParameterControls.svelte';
	import WhatIfPanel from '$lib/components/calcolatore/WhatIfPanel.svelte';
	import ProjectionChart from '$lib/components/calcolatore/ProjectionChart.svelte';
	import ProjectionTable from '$lib/components/calcolatore/ProjectionTable.svelte';
	import TaxOptimization from '$lib/components/calcolatore/TaxOptimization.svelte';
	import PresetSelector from '$lib/components/calcolatore/PresetSelector.svelte';

	// === State ===
	let loading = $state(true);
	let profile = $state<Profile | undefined>(undefined);

	// Adjustable parameters
	let swr = $state(4);
	let expectedReturn = $state(7);
	let inflationRate = $state(2);
	let taxMode = $state<'stocks' | 'btp' | 'blended'>('blended');
	let withdrawalStrategy = $state<'fixed' | 'vpw' | 'guyton-klinger' | 'cape-based' | 'amortized'>(
		'fixed'
	);
	let targetBequest = $state(0);

	// === What-If overrides (initialized from profile in onMount) ===
	let wiAnnualExpenses = $state(20000);
	let wiInitialPortfolio = $state(0);
	let wiAnnualContribution = $state(0);
	let wiRetirementAge = $state(65);
	let wiPensionAmount = $state(0);
	let wiPensionAge = $state(67);
	let wiLifeExpectancy = $state(90);
	let wiOtherIncome = $state(0);
	let wiOtherIncomeEndAge = $state(90);

	// Set ipotesi fiscali attivo (default 2026, il profilo puo' suggerire altro via assumptionsId)
	let assumptions = $state<AssumptionSet>(DEFAULT_2026);

	// Profile defaults for reset & change detection
	let wiDefaults = $state({
		annualExpenses: 20000,
		initialPortfolio: 0,
		annualContribution: 0,
		retirementAge: 65,
		pensionAmount: 0,
		pensionAge: 67,
		lifeExpectancy: 90,
		otherIncome: 0,
		otherIncomeEndAge: 90
	});

	function resetWhatIf() {
		wiAnnualExpenses = wiDefaults.annualExpenses;
		wiInitialPortfolio = wiDefaults.initialPortfolio;
		wiAnnualContribution = wiDefaults.annualContribution;
		wiRetirementAge = wiDefaults.retirementAge;
		wiPensionAmount = wiDefaults.pensionAmount;
		wiPensionAge = wiDefaults.pensionAge;
		wiLifeExpectancy = wiDefaults.lifeExpectancy;
		wiOtherIncome = wiDefaults.otherIncome;
		wiOtherIncomeEndAge = wiDefaults.otherIncomeEndAge;
	}

	// === Derived values ===
	// "blended" = aliquota derivata dalla composizione reale del portafoglio
	// (BFP 12,5%, cripto 33%, resto 26%) invece di un flat 0.20; "stocks"/"btp"
	// restano override manuali, ora agganciati all'AssumptionSet attivo.
	let taxRate = $derived(
		taxMode === 'stocks'
			? assumptions.capital.stocksAndEtf
			: taxMode === 'btp'
				? assumptions.capital.governmentBonds
				: blendedCapitalGainsRate(
						(profile?.portfolio ?? {}) as unknown as Record<string, number>,
						assumptions.capital
					)
	);

	let annualExpenses = $derived(wiAnnualExpenses);
	let withdrawalRate = $derived(swr / 100);

	// Issue #2: opzione per STIMARE il rendimento atteso dall'allocazione
	// (rend. azioni% x quota azioni + rend. obbligazioni% x quota obbligazioni),
	// come i calcolatori FIRE che separano Stock/Bond returns. Default OFF =
	// usa il rendimento atteso singolo impostato a mano (expectedReturn).
	let deriveReturn = $state(false);
	let stockReturnPct = $state(8.0);
	let bondReturnPct = $state(3.5);
	let equityAllocPct = $state(70);
	// La quota azioni (input number) puo' arrivare come stringa o vuota: coerciamo
	// con Number(x)||0 e clampiamo a [0,100] per evitare rendimenti derivati assurdi.
	let eqAllocClamped = $derived(Math.min(100, Math.max(0, Number(equityAllocPct) || 0)));
	let effectiveReturn = $derived.by(() => {
		if (!deriveReturn) return expectedReturn;
		const st = Number(stockReturnPct) || 0;
		const bo = Number(bondReturnPct) || 0;
		return (eqAllocClamped / 100) * st + (1 - eqAllocClamped / 100) * bo;
	});

	// FIRE number tiene conto della pensione INPS, del "ponte" prima di riceverla
	// e di eventuali altri redditi perpetui (affitti, dividendi, rendite)
	let annualPensionIncome = $derived(wiPensionAmount * 13);
	// Issue #2: opzione per ESCLUDERE pensione INPS e altri redditi dal calcolo
	// FIRE (modalita' "solo patrimonio"). Default ON = comportamento attuale.
	let includeFutureIncome = $state(true);
	let effPension = $derived(includeFutureIncome ? annualPensionIncome : 0);
	let effOtherIncome = $derived(includeFutureIncome ? wiOtherIncome : 0);
	let realReturn = $derived((1 + effectiveReturn / 100) / (1 + inflationRate / 100) - 1);
	let fireNumberClassic = $derived(calculateFireNumber(annualExpenses, withdrawalRate));
	let fireNumber = $derived(
		calculateFireNumberWithPension({
			annualExpenses,
			withdrawalRate,
			annualPension: effPension,
			retirementAge: wiRetirementAge,
			pensionAge: wiPensionAge,
			lifeExpectancy: wiLifeExpectancy,
			realReturn,
			otherIncome: effOtherIncome,
			otherIncomeEndAge: wiOtherIncomeEndAge,
			children: profile?.children,
			mortgage: profile?.mortgage,
			baseYear: new Date().getFullYear(),
			currentAge: profile ? new Date().getFullYear() - profile.birthYear : 30
		})
	);
	let bridgeYears = $derived(Math.max(0, wiPensionAge - wiRetirementAge));
	let hasPensionBridge = $derived(effPension > 0 && bridgeYears > 0);

	// Patrimonio totale (info), liquido (usato per FIRE) e illiquido (immobili/TFR)
	let netWorth = $derived(profile ? calculateNetWorth(profile.portfolio as unknown as Record<string, number>) + (profile.spouse?.initialPortfolio || 0) : 0);
	let liquidNetWorth = $derived(profile ? calculateLiquidNetWorth(profile.portfolio as unknown as Record<string, number>) + (profile.spouse?.initialPortfolio || 0) : 0);
	// Quota BFP (esente da bollo titoli) sul liquido, per il calcolo patrimoniale.
	let bolloExemptShare = $derived(
		liquidNetWorth > 0 ? (profile?.portfolio?.bfp ?? 0) / liquidNetWorth : 0
	);
	let illiquidNetWorth = $derived(profile ? calculateIlliquidNetWorth(profile.portfolio as unknown as Record<string, number>) : 0);

	let annualSavings = $derived(wiAnnualContribution);

	let yearsToFire = $derived(
		calculateYearsToFire(wiInitialPortfolio, annualSavings, effectiveReturn / 100, fireNumber, inflationRate / 100)
	);

	let savingsRate = $derived(
		profile
			? calculateSavingsRate(
					(profile.annualIncome || 0) + (profile.otherIncome || 0),
					profile.annualExpenses
				)
			: 0
	);

	let currentAge = $derived(
		profile ? new Date().getFullYear() - profile.birthYear : 30
	);

	let coastFireNumber = $derived(
		profile
			? calculateCoastFireNumber(
					currentAge,
					wiRetirementAge,
					fireNumber,
					effectiveReturn / 100,
					inflationRate / 100
				)
			: 0
	);

	let gap = $derived(fireNumber - wiInitialPortfolio);

	// Trasparenza: il FIRE puo' risultare "troppo presto" per ipotesi ottimistiche
	// (fondo pensione contato come liquido, pensione/altri redditi che abbattono
	// il target). Mostriamo un avviso che spiega i driver quando capita.
	let pensionFundAmount = $derived(profile?.portfolio?.pensionFund ?? 0);
	let fireReductionVsClassic = $derived(Math.max(0, fireNumberClassic - fireNumber));
	let fireLooksEarly = $derived(yearsToFire >= 0 && yearsToFire <= 1);

	let retirementAge = $derived(wiRetirementAge);

	let projections = $derived<YearlyProjection[]>(
		profile
			? projectPortfolio({
					initialPortfolio: wiInitialPortfolio,
					annualContribution: annualSavings,
					annualExpenses: annualExpenses,
					expectedReturn: effectiveReturn / 100,
					inflationRate: inflationRate / 100,
					taxRate: taxRate,
					withdrawalRate: withdrawalRate,
					withdrawalStrategy: withdrawalStrategy,
					annualPension: effPension,
					pensionAge: wiPensionAge,
					otherIncome: effOtherIncome,
					otherIncomeEndAge: wiOtherIncomeEndAge,
					currentAge: currentAge,
					retirementAge: retirementAge,
					lifeExpectancy: wiLifeExpectancy,
					startYear: new Date().getFullYear(),
					children: profile.children,
					mortgage: profile.mortgage,
					lifeEvents: profile.lifeEvents,
					spouse: profile.spouse,
					assumptions: assumptions,
					foreignBrokerShare: profile.foreignBrokerShare ?? 0,
					bolloExemptShare: bolloExemptShare,
					glidePathEnabled: profile.glidePathEnabled ?? false,
					glidePathStartEquity: profile.glidePathStartEquity,
					glidePathEndEquity: profile.glidePathEndEquity,
					targetBequest: targetBequest
				})
			: []
	);

	// Parametri baseline per il sensitivity tornado
	let sensitivityParams = $derived(
		profile
			? {
					initialPortfolio: wiInitialPortfolio,
					annualContribution: annualSavings,
					annualExpenses: annualExpenses,
					expectedReturn: effectiveReturn / 100,
					inflationRate: inflationRate / 100,
					taxRate: taxRate,
					withdrawalRate: withdrawalRate,
					withdrawalStrategy: withdrawalStrategy,
					annualPension: effPension,
					pensionAge: wiPensionAge,
					otherIncome: effOtherIncome,
					otherIncomeEndAge: wiOtherIncomeEndAge,
					currentAge: currentAge,
					retirementAge: retirementAge,
					lifeExpectancy: wiLifeExpectancy,
					startYear: new Date().getFullYear(),
					children: profile.children,
					mortgage: profile.mortgage,
					lifeEvents: profile.lifeEvents,
					spouse: profile.spouse,
					assumptions: assumptions,
					foreignBrokerShare: profile.foreignBrokerShare ?? 0,
					bolloExemptShare: bolloExemptShare,
					glidePathEnabled: profile.glidePathEnabled ?? false
				}
			: null
	);

	let withdrawalPlans = $derived<WithdrawalPlan[]>(
		profile
			? optimizeWithdrawalOrder([
					{ type: 'taxable', balance: (profile.portfolio.stocks || 0) + (profile.portfolio.bonds || 0) + (profile.portfolio.crypto || 0) + (profile.portfolio.gold || 0) },
					{ type: 'pension_fund', balance: profile.portfolio.pensionFund || 0 },
					{ type: 'government_bonds', balance: profile.portfolio.cash || 0 },
					{ type: 'tfr', balance: profile.portfolio.tfr || 0 }
				])
			: []
	);

	// === Helper: initialize What-If state from profile ===
	function initWhatIfFromProfile(p: Profile) {
		// Usiamo il patrimonio LIQUIDO come base per il FIRE (immobili e TFR
		// non sono prelevabili al 4%).
		const liquidNw = calculateLiquidNetWorth(p.portfolio as unknown as Record<string, number>);
		const monthlyTotal = Object.values(p.monthlyContributions).reduce((s, v) => s + (v || 0), 0) * 12;
		const income = (p.annualIncome || 0) + (p.otherIncome || 0);
		// I contributi non possono superare il reddito disponibile
		const contribution = income > 0
			? Math.min(monthlyTotal, Math.max(0, income - (p.annualExpenses || 0)))
			: 0;

		const lifeExp = p.lifeExpectancy || 90;
		const defaults = {
			annualExpenses: p.fireExpenses || p.annualExpenses || 20000,
			initialPortfolio: liquidNw + (p.spouse?.initialPortfolio || 0),
			annualContribution: contribution,
			retirementAge: p.retirementAge || 65,
			pensionAmount: p.pension?.estimatedMonthly || 0,
			pensionAge: p.pension?.pensionAge || 67,
			lifeExpectancy: lifeExp,
			otherIncome: p.otherIncome || 0,
			otherIncomeEndAge: p.otherIncomeEndAge ?? lifeExp
		};

		wiDefaults = { ...defaults };
		wiAnnualExpenses = defaults.annualExpenses;
		wiInitialPortfolio = defaults.initialPortfolio;
		wiAnnualContribution = defaults.annualContribution;
		wiRetirementAge = defaults.retirementAge;
		wiPensionAmount = defaults.pensionAmount;
		wiPensionAge = defaults.pensionAge;
		wiLifeExpectancy = defaults.lifeExpectancy;
		wiOtherIncome = defaults.otherIncome;
		wiOtherIncomeEndAge = defaults.otherIncomeEndAge;
	}

	// === Load profile ===
	onMount(async () => {
		try {
			const profiles = await getAllProfiles();
			if (profiles.length > 0) {
				profile = profiles[0];
				// Initialize params from profile
				if (profile.simulation) {
					swr = (profile.simulation.withdrawalRate || 0.04) * 100;
					expectedReturn = (profile.simulation.expectedReturn || 0.07) * 100;
					inflationRate = (profile.simulation.inflationRate || 0.02) * 100;
					const strat = profile.simulation.withdrawalStrategy;
					if (strat === 'fixed') withdrawalStrategy = 'fixed';
					else if (strat === 'vpw') withdrawalStrategy = 'vpw';
					else if (strat === 'guyton-klinger') withdrawalStrategy = 'guyton-klinger';
					else if (strat === 'cape-based') withdrawalStrategy = 'cape-based';
					else if (strat === 'amortized') withdrawalStrategy = 'amortized';
					targetBequest = profile.simulation.targetBequest ?? 0;
				}
				initWhatIfFromProfile(profile);
			}
		} catch (err) {
			console.error('Errore caricamento profilo:', err);
		} finally {
			loading = false;
		}
	});
</script>

<svelte:head>
	<title>Calcolatore FIRE - FIRE Planner</title>
</svelte:head>

<Breadcrumb class="mb-4">
	<BreadcrumbItem href="/" home>Dashboard</BreadcrumbItem>
	<BreadcrumbItem>Calcolatore FIRE</BreadcrumbItem>
</Breadcrumb>

<Heading tag="h1" class="mb-2">Calcolatore FIRE</Heading>
<p class="text-gray-600 dark:text-gray-400 mb-8">
	Calcola il tuo numero FIRE, il tasso di risparmio e gli anni necessari per raggiungere
	l'indipendenza finanziaria in base ai tuoi dati.
</p>

{#if loading}
	<div class="flex items-center justify-center py-20">
		<Spinner size="12" />
		<span class="ml-3 text-gray-500 dark:text-gray-400">Caricamento profilo...</span>
	</div>
{:else if !profile}
	<Alert color="yellow" class="max-w-2xl">
		{#snippet icon()}
			<InfoCircleSolid class="w-5 h-5" />
		{/snippet}
		<span class="font-medium">Nessun profilo trovato.</span>
		Per utilizzare il calcolatore FIRE devi prima compilare il tuo profilo finanziario.
		<a
			href="/profilo/"
			class="inline-flex items-center ml-2 font-semibold text-yellow-800 dark:text-yellow-300 underline hover:no-underline"
		>
			Vai al Profilo
		</a>
	</Alert>
{:else}
	<!-- Pannello "Ipotesi attive" sempre visibile sopra ai tab: rende il calcolo verificabile -->
	<AssumptionsPanel bind:assumptions />

	<Tabs tabStyle="underline" classes={{ content: 'p-0 mt-6' }}>
		<TabItem open>
			{#snippet titleSlot()}
				<div class="flex items-center gap-2">
					<ChartPieOutline class="w-4 h-4" />
					Panoramica
				</div>
			{/snippet}

			<!-- FIRE Number Hero -->
			<div class="mb-6">
				<FireHero
					{fireNumber}
					{annualExpenses}
					{withdrawalRate}
					annualPension={effPension}
					{fireNumberClassic}
					{hasPensionBridge}
					{bridgeYears}
					pensionAge={wiPensionAge}
					retirementAge={wiRetirementAge}
					otherIncome={effOtherIncome}
					otherIncomeEndAge={wiOtherIncomeEndAge}
				/>
			</div>

			<!-- Issue #2: opzione "solo patrimonio" (escludi pensione/altri redditi dal FIRE) -->
			<div class="mb-6 rounded-lg bg-gray-50 dark:bg-gray-800/60 p-4 border border-gray-200 dark:border-gray-700">
				<Toggle bind:checked={includeFutureIncome}>
					Considera pensione INPS e altri redditi nel FIRE Number
				</Toggle>
				<Helper class="mt-1 text-xs">
					Se disattivato, il FIRE Number e la proiezione si basano <strong>solo sul patrimonio</strong>
					(modalita' conservativa): pensione INPS e altri redditi non riducono il capitale necessario.
				</Helper>
			</div>

			<!-- Key Metrics -->
			<MetricsGrid
				{fireNumber}
				{yearsToFire}
				{savingsRate}
				{netWorth}
				{liquidNetWorth}
				{illiquidNetWorth}
				{coastFireNumber}
				{gap}
			/>

			<!-- Varianti FIRE (Lean/Standard/Barista/Chubby/Fat) sui propri numeri -->
			<FireVariants {annualExpenses} {withdrawalRate} {liquidNetWorth} />

			{#if fireLooksEarly}
				<Alert color="yellow" class="mt-6">
					{#snippet icon()}
						<InfoCircleSolid class="w-5 h-5" />
					{/snippet}
					<span class="font-medium">FIRE molto vicino: controlla le ipotesi.</span>
					<p class="mt-2 text-sm">
						Il risultato indica che sei (quasi) gia' FIRE. Spesso dipende da ipotesi
						ottimistiche che vale la pena verificare:
					</p>
					<ul class="mt-2 ml-4 list-disc text-sm space-y-1">
						{#if pensionFundAmount > 0}
							<li>
								Il patrimonio di partenza usato per il calcolo ({formatCurrency(wiInitialPortfolio)})
								comprende, nel dato di profilo, il <strong>fondo pensione</strong>
								({formatCurrency(pensionFundAmount)}), prelevabile
								pero' solo via <strong>RITA</strong> in prossimita' della pensione: oggi non e'
								davvero disponibile per vivere di rendita.
							</li>
						{/if}
						{#if effPension > 0}
							<li>
								Il FIRE number e' ridotto dal valore attuale della <strong>pensione INPS</strong>
								(~{formatCurrency(effPension)}/anno dai {wiPensionAge} anni).
							</li>
						{/if}
						{#if effOtherIncome > 0}
							<li>
								Sono conteggiati <strong>altri redditi</strong> ({formatCurrency(effOtherIncome)}/anno)
								come rendita che abbassa il capitale necessario.
							</li>
						{/if}
						{#if fireReductionVsClassic > 0}
							<li>
								Senza questi aiuti il FIRE number "classico" (spese / SWR) sarebbe
								<strong>{formatCurrency(fireNumberClassic)}</strong>, non {formatCurrency(fireNumber)}.
							</li>
						{/if}
					</ul>
					<p class="mt-2 text-sm">
						Apri la <strong>Simulazione Monte Carlo</strong> per testare la robustezza del piano
						contro i ribassi di mercato.
					</p>
				</Alert>
			{/if}
		</TabItem>

		<TabItem>
			{#snippet titleSlot()}
				<div class="flex items-center gap-2">
					<ChartMixedDollarSolid class="w-4 h-4" />
					Simulazione
				</div>
			{/snippet}

			<!-- Preset Modes -->
			<div class="mb-6">
				<PresetSelector bind:swr bind:expectedReturn bind:inflationRate />
			</div>

			<!-- Controls Row: Strategy + Parameters -->
			<div class="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-6">
				<WithdrawalStrategySelector bind:selected={withdrawalStrategy} bind:targetBequest />
				<ParameterControls bind:swr bind:expectedReturn bind:inflationRate bind:taxMode returnOverridden={deriveReturn} />
			</div>

			<!-- Issue #2: stima del rendimento atteso dall'allocazione azioni/obbligazioni -->
			<div class="mb-6 rounded-lg bg-blue-50 dark:bg-blue-900/20 p-4 border border-blue-200 dark:border-blue-800">
				<Toggle bind:checked={deriveReturn}>
					Stima il rendimento atteso dall'allocazione
				</Toggle>
				<Helper class="mt-1 mb-3 text-xs">
					Invece di un rendimento unico, ricava il rendimento atteso da rendimento azioni/obbligazioni
					e dalla quota di ciascuno (come i calcolatori che separano Stock/Bond returns). Quando attivo,
					sostituisce il "Rendimento atteso" impostato sopra.
				</Helper>
				{#if deriveReturn}
					<div class="grid grid-cols-1 sm:grid-cols-3 gap-3">
						<div>
							<Label for="stock-ret-calc" class="mb-1 text-xs">Rendimento azioni (%)</Label>
							<Input id="stock-ret-calc" type="number" bind:value={stockReturnPct} step={0.1} size="sm" />
						</div>
						<div>
							<Label for="bond-ret-calc" class="mb-1 text-xs">Rendimento obbligazioni (%)</Label>
							<Input id="bond-ret-calc" type="number" bind:value={bondReturnPct} step={0.1} size="sm" />
						</div>
						<div>
							<Label for="equity-alloc-calc" class="mb-1 text-xs">Quota azioni (%)</Label>
							<Input id="equity-alloc-calc" type="number" bind:value={equityAllocPct} min={0} max={100} step={5} size="sm" />
						</div>
					</div>
					<p class="mt-3 text-sm text-blue-800 dark:text-blue-300">
						Rendimento atteso derivato: <strong>{effectiveReturn.toFixed(2)}%</strong>
						({eqAllocClamped}% azioni / {100 - eqAllocClamped}% obbligazioni)
					</p>
				{/if}
			</div>

			<!-- What-If Panel -->
			<div class="mb-6">
				<WhatIfPanel
					bind:annualExpenses={wiAnnualExpenses}
					bind:initialPortfolio={wiInitialPortfolio}
					bind:annualContribution={wiAnnualContribution}
					bind:retirementAge={wiRetirementAge}
					bind:pensionAmount={wiPensionAmount}
					bind:pensionAge={wiPensionAge}
					bind:lifeExpectancy={wiLifeExpectancy}
					bind:otherIncome={wiOtherIncome}
					bind:otherIncomeEndAge={wiOtherIncomeEndAge}
					defaults={wiDefaults}
					onreset={resetWhatIf}
				/>
			</div>

			<!-- Sensitivity Analysis: tornado chart sulle leve principali -->
			{#if sensitivityParams}
				<div class="mb-6">
					<SensitivityTornado baseParams={sensitivityParams} shock={0.1} metric="finalPortfolio" />
					<p class="mt-3 text-xs text-gray-500 dark:text-gray-400 leading-relaxed">
						<strong>Cosa mostra:</strong> l'<strong>analisi di sensitività</strong> (tornado chart) del piano FIRE.
						Ogni variabile di input (es. rendimento atteso, tasso prelievo, inflazione, spese, contributi) 
						è shockata di <strong>±10%</strong> rispetto al baseline.
						Le barre <span class="text-red-600 dark:text-red-400 font-medium">rosse</span> mostrano l'impatto di uno shock negativo (−10%), 
						quelle <span class="text-green-600 dark:text-green-400 font-medium">verdi</span> uno shock positivo (+10%).
						Le variabili sono ordinate per impatto assoluto (più influenti in alto).
					</p>
					<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">
						<strong>A cosa serve:</strong> identificare le <strong>leve critiche</strong> del piano.
						Se il "Rendimento atteso" ha la barra più lunga, piccole variazioni di mercato cambiano molto il risultato finale → 
						serve allocazione più conservativa o margine di sicurezza maggiore.
						Passa il mouse sulle barre per vedere i valori esatti.
					</p>
				</div>
			{/if}

			<!-- Projection Chart -->
			<div>
				<ProjectionChart
					{projections}
					{fireNumber}
					{retirementAge}
					{currentAge}
					inflationRate={inflationRate / 100}
				/>
				<p class="mt-3 text-xs text-gray-500 dark:text-gray-400 leading-relaxed">
					<strong>Cosa mostra:</strong> la <strong>proiezione deterministica</strong> del portafoglio anno per anno.
					La linea <span class="text-green-600 dark:text-green-400 font-medium">verde</span> = fase di <strong>accumulo</strong> (contributi + rendimenti composti), 
					la linea <span class="text-amber-600 dark:text-amber-400 font-medium">arancione</span> = fase di <strong>decumulo</strong> (prelievi per spese).
					La linea tratteggiata <span class="text-red-600 dark:text-red-400 font-medium">rossa</span> = <strong>Target FIRE</strong> (in euro correnti, cresce con l'inflazione).
					Il tooltip mostra dettagli: portafoglio, target, contributi, prelievi, rendimenti, tasse.
				</p>
				<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">
					<strong>A cosa serve:</strong> visualizzare il percorso "caso base" senza alea. 
					Utile per pianificare i flussi di cassa, verificare quando si raggiunge il FIRE Number 
					e stimare la durata del patrimonio in decumulo. 
					Attiva "Stima rendimento da allocazione" per un rendimento derivato da azioni/obbligazioni.
				</p>
			</div>
		</TabItem>

		<TabItem>
			{#snippet titleSlot()}
				<div class="flex items-center gap-2">
					<TableColumnSolid class="w-4 h-4" />
					Cash flow
				</div>
			{/snippet}

			<ProjectionTable
				{projections}
				{retirementAge}
				pensionAge={wiPensionAge}
			/>
		</TabItem>

		<TabItem>
			{#snippet titleSlot()}
				<div class="flex items-center gap-2">
					<CashSolid class="w-4 h-4" />
					Prelievi
				</div>
			{/snippet}

			<TaxOptimization plans={withdrawalPlans} />
		</TabItem>
	</Tabs>
{/if}
