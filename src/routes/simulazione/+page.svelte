<script lang="ts">
	import { onMount } from 'svelte';
	import {
		Heading,
		Breadcrumb,
		BreadcrumbItem,
		Button,
		Toast
	} from 'flowbite-svelte';
	import {
		FloppyDiskSolid,
		CheckCircleSolid,
		CloseCircleSolid
	} from 'flowbite-svelte-icons';
	import type { MonteCarloParams, MonteCarloResult } from '$lib/engine/monte-carlo';
	import type { SimulationResult } from '$lib/db/index';
	import { goldReturns } from '$lib/data/gold';
	import {
		ALL_SERIES_PRESETS,
		getSeriesPreset,
		type SeriesPresetId
	} from '$lib/data/default-series';
	import { DEFAULT_2026, getPreset } from '$lib/engine/assumptions';
	import { blendedCapitalGainsRate } from '$lib/engine/tax-italy';
	import {
		createResult,
		getAllResults,
		deleteResult
	} from '$lib/db/results';
	import { getAllProfiles } from '$lib/db/profiles';
	import { calculateLiquidNetWorth } from '$lib/engine/fire-calculator';

	import ConfigPanel from '$lib/components/simulazione/ConfigPanel.svelte';
	import SuccessGauge from '$lib/components/simulazione/SuccessGauge.svelte';
	import FanChart from '$lib/components/simulazione/FanChart.svelte';
	import HistogramChart from '$lib/components/simulazione/HistogramChart.svelte';
	import SummaryTable from '$lib/components/simulazione/SummaryTable.svelte';
	import ScenarioTabs from '$lib/components/simulazione/ScenarioTabs.svelte';
	import PreviousResults from '$lib/components/simulazione/PreviousResults.svelte';

	// State
	let running = $state(false);
	let progress = $state(0);
	let elapsedTime = $state(0);
	let result = $state<MonteCarloResult | null>(null);
	let previousResults = $state<SimulationResult[]>([]);
	let lastParams = $state<Partial<MonteCarloParams> | null>(null);
	let toastMessage = $state('');
	let toastType = $state<'success' | 'error'>('success');
	let showToast = $state(false);
	let profileLoaded = $state(false);
	// Parametri fiscali del MC derivati dal profilo (coerenza col deterministico):
	// aliquota capital gain blended dalla composizione e quota BFP esente da bollo.
	let mcCapGainsRate = $state(0);
	let mcBolloExemptShare = $state(0);
	// #5: valori iniziali della simulazione ereditati dal profilo (come il calcolatore).
	let simDefaults = $state<{
		initialPortfolio?: number; annualContribution?: number; annualExpenses?: number;
		yearsToFire?: number; yearsInRetirement?: number; currentAge?: number;
		pensionAnnual?: number; pensionStartAge?: number; otherIncomeAnnual?: number; otherIncomeEndAge?: number;
	}>({});
	let paramsReady = $state(false);

	// Worker reference
	let worker: Worker | null = null;
	let timerInterval: ReturnType<typeof setInterval> | null = null;

	// Historical returns as decimals - default IT-centric (MSCI World + BTP + ISTAT).
	// L'utente puo' switchare al preset US classico (S&P 500 + bond US + CPI USA).
	let seriesPresetId = $state<SeriesPresetId>('it-default');
	let historicalStockReturns = $derived(getSeriesPreset(seriesPresetId).stocks);
	let historicalBondReturns = $derived(getSeriesPreset(seriesPresetId).bonds);
	const historicalGoldReturns = goldReturns.map((r) => r.value / 100);
	let historicalInflation = $derived(getSeriesPreset(seriesPresetId).inflation);

	onMount(() => {
		loadPreviousResults();
		loadProfileDefaults();
		return () => {
			worker?.terminate();
			if (timerInterval) clearInterval(timerInterval);
		};
	});

	async function loadPreviousResults() {
		try {
			previousResults = await getAllResults();
		} catch {
			// IndexedDB may not be available
		}
	}

	async function loadProfileDefaults() {
		try {
			const profiles = await getAllProfiles();
			if (profiles.length > 0) {
				const p = profiles[0];
				const liquid = calculateLiquidNetWorth(p.portfolio as unknown as Record<string, number>);
				mcCapGainsRate = blendedCapitalGainsRate(
					p.portfolio as unknown as Record<string, number>,
					getPreset(p.assumptionsId).capital
				);
				mcBolloExemptShare = liquid > 0 ? (p.portfolio.bfp ?? 0) / liquid : 0;
				const monthly =
					Object.values(p.monthlyContributions).reduce((s, v) => s + (v || 0), 0) * 12;
				const income = (p.annualIncome || 0) + (p.otherIncome || 0);
				const contribution =
					income > 0 ? Math.min(monthly, Math.max(0, income - (p.annualExpenses || 0))) : monthly;
				const age = new Date().getFullYear() - p.birthYear;
				const retAge = p.retirementAge || 65;
				const lifeExp = p.lifeExpectancy || 90;
				simDefaults = {
					initialPortfolio: Math.round(liquid),
					annualContribution: Math.round(contribution),
					annualExpenses: Math.round(p.fireExpenses || p.annualExpenses || 30000),
					yearsToFire: Math.max(0, retAge - age),
					yearsInRetirement: Math.max(1, lifeExp - retAge),
					currentAge: age,
					pensionAnnual: Math.round((p.pension?.estimatedMonthly || 0) * 13),
					pensionStartAge: p.pension?.pensionAge || 67,
					otherIncomeAnnual: Math.round(p.otherIncome || 0),
					otherIncomeEndAge: p.otherIncomeEndAge ?? lifeExp
				};
				profileLoaded = true;
			}
		} catch {
			/* nessun profilo: si usano i default */
		} finally {
			paramsReady = true;
		}
	}

	function showNotification(message: string, type: 'success' | 'error' = 'success') {
		toastMessage = message;
		toastType = type;
		showToast = true;
		setTimeout(() => {
			showToast = false;
		}, 4000);
	}

	function startTimer() {
		const startTime = performance.now();
		elapsedTime = 0;
		timerInterval = setInterval(() => {
			elapsedTime = Math.round(performance.now() - startTime);
		}, 100);
	}

	function stopTimer() {
		if (timerInterval) {
			clearInterval(timerInterval);
			timerInterval = null;
		}
	}

	function handleRun(params: Partial<MonteCarloParams>) {
		if (running) return;

		running = true;
		progress = 0;
		result = null;
		lastParams = params;
		startTimer();

		// Add historical data for non-parametric modes
		if (params.simulationMode !== 'parametric') {
			params.historicalStockReturns = historicalStockReturns;
			params.historicalBondReturns = historicalBondReturns;
			params.historicalInflation = historicalInflation;

			// Attach historical returns to multi-asset configs
			if (params.assetClasses) {
				const histMap: Record<string, number[]> = {
					stocks: historicalStockReturns,
					bonds: historicalBondReturns,
					gold: historicalGoldReturns
				};
				for (const ac of params.assetClasses) {
					if (histMap[ac.name]) {
						ac.historicalReturns = histMap[ac.name];
					}
				}
			}
		}

		// Terminate previous worker if still running
		worker?.terminate();

		try {
			worker = new Worker(
				new URL('$lib/workers/monte-carlo.worker.ts', import.meta.url),
				{ type: 'module' }
			);

			worker.onmessage = (e: MessageEvent) => {
				const data = e.data;
				if (data.type === 'progress') {
					progress = data.percent;
				} else if (data.type === 'result') {
					result = data.data as MonteCarloResult;
					running = false;
					progress = 100;
					stopTimer();
					showNotification('Simulazione completata con successo!');
				} else if (data.type === 'error') {
					running = false;
					stopTimer();
					showNotification(`Errore: ${data.message}`, 'error');
				}
			};

			worker.onerror = (e) => {
				running = false;
				stopTimer();
				showNotification(`Errore nel worker: ${e.message}`, 'error');
			};

			worker.postMessage({
				type: 'run',
				params: {
					initialPortfolio: params.initialPortfolio ?? 200000,
					annualContribution: params.annualContribution ?? 20000,
					annualExpenses: params.annualExpenses ?? 30000,
					withdrawalRate: params.withdrawalRate ?? 0.04,
					withdrawalStrategy: params.withdrawalStrategy ?? 'fixed',
					stockAllocation: params.stockAllocation ?? 0.7,
					bondAllocation: params.bondAllocation ?? 0.3,
					yearsToFire: params.yearsToFire ?? 10,
					yearsInRetirement: params.yearsInRetirement ?? 30,
					inflationRate: params.inflationRate ?? 0.02,
					simulationMode: params.simulationMode ?? 'historical',
					iterations: params.iterations ?? 10000,
					expectedStockReturn: params.expectedStockReturn,
					stockStdDev: params.stockStdDev,
					expectedBondReturn: params.expectedBondReturn,
					bondStdDev: params.bondStdDev,
					historicalStockReturns: params.historicalStockReturns,
					historicalBondReturns: params.historicalBondReturns,
					historicalInflation: params.historicalInflation,
					useCorrelation: params.useCorrelation,
					assetClasses: params.assetClasses,
					correlationMatrix: params.correlationMatrix,
					glidePathEnabled: params.glidePathEnabled,
					glidePathStartEquity: params.glidePathStartEquity,
					glidePathEndEquity: params.glidePathEndEquity,
					stampDutyRate: params.stampDutyRate ?? DEFAULT_2026.capital.stampDuty,
					ivafeRate: params.ivafeRate ?? DEFAULT_2026.capital.ivafe,
					foreignBrokerShare: params.foreignBrokerShare ?? 0,
					capitalGainsTaxRate: params.capitalGainsTaxRate ?? mcCapGainsRate,
					bolloExemptShare: params.bolloExemptShare ?? mcBolloExemptShare,
					currentAge: params.currentAge,
					annualPension: params.annualPension ?? 0,
					pensionAge: params.pensionAge,
					otherIncome: params.otherIncome ?? 0,
					otherIncomeEndAge: params.otherIncomeEndAge
				} satisfies MonteCarloParams
			});
		} catch (err) {
			running = false;
			stopTimer();
			showNotification('Impossibile avviare il Web Worker', 'error');
		}
	}

	async function handleSave() {
		if (!result || !lastParams) return;

		try {
			const totalYears = (lastParams.yearsToFire ?? 10) + (lastParams.yearsInRetirement ?? 30);
			const finalYear = result.percentiles.p5.length - 1;

			await createResult({
				scenarioId: 0,
				profileId: 0,
				iterations: lastParams.iterations ?? 10000,
				years: totalYears,
				successRate: result.successRate,
				medianFinalValue: result.medianFinalValue,
				percentiles: {
					p5: result.percentiles.p5[finalYear] ?? 0,
					p10: result.percentiles.p10[finalYear] ?? 0,
					p25: result.percentiles.p25[finalYear] ?? 0,
					p50: result.percentiles.p50[finalYear] ?? 0,
					p75: result.percentiles.p75[finalYear] ?? 0,
					p90: result.percentiles.p90[finalYear] ?? 0,
					p95: result.percentiles.p95[finalYear] ?? 0
				},
				yearlyData: result!.yearlyStats.map((s) => ({
					year: s.year,
					median: s.median,
					p5: s.p5,
					p25: result!.percentiles.p25[s.year - 1] ?? 0,
					p75: result!.percentiles.p75[s.year - 1] ?? 0,
					p95: s.p95,
					successRate: s.successRate
				})),
				params: {
					withdrawalRate: (lastParams.withdrawalRate ?? 0.04) * 100,
					withdrawalStrategy: 'fixed',
					stockAllocation: (lastParams.stockAllocation ?? 0.7) * 100,
					bondAllocation: (lastParams.bondAllocation ?? 0.3) * 100,
					glidePathEnabled: false,
					inflationRate: (lastParams.inflationRate ?? 0.02) * 100,
					expectedReturn: 7,
					iterations: lastParams.iterations ?? 10000
				}
			});

			await loadPreviousResults();
			showNotification('Risultato salvato con successo!');
		} catch (err) {
			showNotification('Errore nel salvataggio', 'error');
		}
	}

	function handleLoadResult(saved: SimulationResult) {
		// Reconstruct a MonteCarloResult from the saved data
		const p5: number[] = saved.yearlyData.map((d) => d.p5);
		const p25: number[] = saved.yearlyData.map((d) => d.p25);
		const p50: number[] = saved.yearlyData.map((d) => d.median);
		const p75: number[] = saved.yearlyData.map((d) => d.p75);
		const p95: number[] = saved.yearlyData.map((d) => d.p95);
		// Approximate p10 and p90
		const p10: number[] = saved.yearlyData.map((d) => Math.round((d.p5 + d.median) / 2));
		const p90: number[] = saved.yearlyData.map((d) => Math.round((d.median + d.p95) / 2));

		result = {
			successRate: saved.successRate,
			medianFinalValue: saved.medianFinalValue,
			meanFinalValue: saved.medianFinalValue,
			percentiles: { p5, p10, p25, p50, p75, p90, p95 },
			yearlyStats: saved.yearlyData.map((d) => ({
				year: d.year,
				median: d.median,
				mean: d.median,
				p5: d.p5,
				p95: d.p95,
				successRate: d.successRate
			})),
			worstCase: p5,
			bestCase: p95,
			failureYear: null,
			// Un risultato salvato non porta con se' l'errore standard: lo lasciamo
			// neutro (nessuna banda di incertezza ricostruibile a posteriori).
			successRateStdError: 0,
			successRateCI95: { lower: saved.successRate, upper: saved.successRate }
		};

		showNotification('Risultato caricato');
	}

	async function handleDeleteResult(id: number) {
		try {
			await deleteResult(id);
			await loadPreviousResults();
			showNotification('Risultato eliminato');
		} catch {
			showNotification('Errore nella cancellazione', 'error');
		}
	}
</script>

<svelte:head>
	<title>Simulazione Monte Carlo - FIRE Planner</title>
</svelte:head>

<!-- Toast notification -->
{#if showToast}
	<div class="fixed top-4 right-4 z-50">
		<Toast color={toastType === 'success' ? 'green' : 'red'} dismissable toastStatus={showToast}>
			{#snippet icon()}
				{#if toastType === 'success'}
					<CheckCircleSolid class="w-5 h-5" />
				{:else}
					<CloseCircleSolid class="w-5 h-5" />
				{/if}
			{/snippet}
			{toastMessage}
		</Toast>
	</div>
{/if}

<Breadcrumb class="mb-4">
	<BreadcrumbItem href="/" home>Dashboard</BreadcrumbItem>
	<BreadcrumbItem>Simulazione Monte Carlo</BreadcrumbItem>
</Breadcrumb>

<div class="flex items-start justify-between mb-6 flex-wrap gap-4">
	<div>
		<Heading tag="h1" class="mb-2">Simulazione Monte Carlo</Heading>
		<p class="text-gray-600 dark:text-gray-400">
			Testa la robustezza del tuo piano FIRE simulando migliaia di scenari di mercato.
		</p>
	</div>
	{#if result && !running}
		<Button color="primary" onclick={handleSave}>
			<FloppyDiskSolid class="w-4 h-4 me-2" />
			Salva Risultato
		</Button>
	{/if}
</div>

<!-- Series preset switcher -->
<div class="mb-4 p-4 rounded-lg bg-blue-50 dark:bg-blue-950/30 border border-blue-200 dark:border-blue-800">
	<div class="flex flex-col sm:flex-row sm:items-center gap-3">
		<div class="flex-1">
			<label for="series-preset" class="text-sm font-semibold text-gray-700 dark:text-gray-200 block mb-1">
				Serie storica di riferimento
			</label>
			<p class="text-xs text-gray-600 dark:text-gray-400">
				{getSeriesPreset(seriesPresetId).description}
			</p>
		</div>
		<select
			id="series-preset"
			bind:value={seriesPresetId}
			class="rounded-md border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-700 px-3 py-2 text-sm"
		>
			{#each ALL_SERIES_PRESETS as preset (preset.id)}
				<option value={preset.id}>{preset.label}</option>
			{/each}
		</select>
	</div>
</div>

<!-- Configuration Panel -->
<div class="mb-8">
	{#if paramsReady}
		<ConfigPanel
			onRun={handleRun}
			{running}
			{progress}
			{elapsedTime}
			{profileLoaded}
			seriesLabel={getSeriesPreset(seriesPresetId).label}
			seriesDescription={getSeriesPreset(seriesPresetId).description}
			defaults={simDefaults}
		/>
	{:else}
		<div class="py-10 text-center text-gray-400 dark:text-gray-500">Caricamento dati dal profilo…</div>
	{/if}
</div>

<!-- Results Section -->
{#if result}
	<div class="space-y-8 mb-8">
		<!-- Animated entrance -->
		<div class="animate-fade-in">
			<!-- Success Gauge + Key Stats -->
			<div class="mb-8">
				<SuccessGauge {result} />
				<p class="mt-3 text-xs text-gray-500 dark:text-gray-400 leading-relaxed">
					<strong>Cosa mostra:</strong> il <strong>tasso di successo</strong> della simulazione (percentuale di scenari in cui il patrimonio non si esaurisce prima della fine dell'orizzonte temporale).
					L'anello colorato indica: <span class="text-green-600 dark:text-green-400 font-medium">verde</span> ≥95% (eccellente), 
					<span class="text-amber-600 dark:text-amber-400 font-medium">giallo</span> 80–94% (buono/sufficiente), 
					<span class="text-red-600 dark:text-red-400 font-medium">rosso</span> sotto 80% (a rischio).
					Sotto: valore finale mediano, media e percentili P5–P95.
				</p>
				<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">
					<strong>A cosa serve:</strong> valutare rapidamente la robustezza del piano. Un tasso sopra 90% è considerato solido;
					se sotto 80% serve rivedere spese, prelievo o allocazione. Usa la Simulazione per testare varianti.
				</p>
			</div>

			<!-- Fan Chart -->
			<div class="mb-8">
				<FanChart {result} />
				<p class="mt-3 text-xs text-gray-500 dark:text-gray-400 leading-relaxed">
					<strong>Cosa mostra:</strong> la <strong>proiezione probabilistica</strong> del portafoglio anno per anno tramite bande di confidenza:
					<span class="text-blue-200 dark:text-blue-800 font-medium">P5–P95</span> (chiara, 90% degli scenari), 
					<span class="text-blue-300 dark:text-blue-700 font-medium">P10–P90</span> (media, 80%), 
					<span class="text-blue-400 dark:text-blue-600 font-medium">P25–P75</span> (scura, 50% centrale), 
					e la <span class="text-blue-600 dark:text-blue-400 font-medium">mediana (P50)</span> (linea blu).
				</p>
				<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">
					<strong>A cosa serve:</strong> vedere l'intervallo di possibili evoluzioni del patrimonio, non solo la media. 
					L'allargamento delle bande nel tempo riflette l'incertezza crescente. 
					Usa lo slider in basso per zoomare su periodi specifici.
				</p>
			</div>

			<!-- Histogram + Summary Table -->
			<div class="grid grid-cols-1 xl:grid-cols-2 gap-8 mb-8">
				<div>
					<HistogramChart {result} />
					<p class="mt-3 text-xs text-gray-500 dark:text-gray-400 leading-relaxed">
						<strong>Cosa mostra:</strong> la <strong>distribuzione dei valori finali</strong> del portafoglio alla fine della simulazione.
						L'istogramma approssima la forma della distribuzione; la barra <span class="text-blue-600 dark:text-blue-400 font-medium">blu scura</span> indica il bin contenente la <strong>mediana</strong>, 
						la linea tratteggiata <span class="text-blue-600 dark:text-blue-400 font-medium">blu</span> la mediana esatta, 
						quella <span class="text-red-600 dark:text-red-400 font-medium">rossa</span> la media.
					</p>
					<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">
						<strong>A cosa serve:</strong> capire la forma della distribuzione (simmetrica, asimmetrica, coda pesante) 
						e la probabilità di risultati molto diversi dalla mediana.
					</p>
				</div>
				<SummaryTable {result} />
			</div>

			<!-- Scenario Detail Tabs -->
			<div class="mb-8">
				<ScenarioTabs {result} />
				<p class="mt-3 text-xs text-gray-500 dark:text-gray-400 leading-relaxed">
					<strong>Cosa mostra:</strong> analisi dettagliate per scenari specifici: 
					<strong>Pessimista (P5)</strong>, <strong>Mediano (P50)</strong>, <strong>Ottimista (P95)</strong>, 
					e <strong>Anno di esaurimento</strong> per gli scenari falliti.
					Ogni tab mostra flusso di cassa, prelievi, tasse e patrimonio anno per anno.
				</p>
				<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">
					<strong>A cosa serve:</strong> prepararsi al "peggior caso plausibile" (P5) e capire 
					quanto margine c'è nello scenario mediano. Utile per pianificare riserve di sicurezza.
				</p>
			</div>
		</div>
	</div>
{/if}

<!-- Previous Results -->
<div class="mb-8">
	<PreviousResults
		results={previousResults}
		onLoad={handleLoadResult}
		onDelete={handleDeleteResult}
	/>
</div>

<style>
	@keyframes fadeIn {
		from {
			opacity: 0;
			transform: translateY(20px);
		}
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	:global(.animate-fade-in) {
		animation: fadeIn 0.6s ease-out;
	}
</style>



