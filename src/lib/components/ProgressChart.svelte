<script lang="ts">
	import { onMount } from 'svelte';
	import { habits } from '../stores/habits';
	import { BarChart3, TrendingUp, Calendar } from 'lucide-svelte';
	import Chart from 'chart.js/auto';
	import type { Chart as ChartType } from 'chart.js';
	import type { Habit } from '../stores/habits';

	let chartCanvas: HTMLCanvasElement | null = null;
	let chartInstance: ChartType | null = null;
	let viewMode: 'weekly' | 'monthly' = 'weekly';

	$: if ($habits && chartCanvas) {
		updateChart();
	}

	onMount(() => {
		if (chartCanvas) {
			createChart();
		}
		return () => {
			if (chartInstance) {
				chartInstance.destroy();
			}
		};
	});

	function createChart() {
		if (!chartCanvas) return;
		const ctx = chartCanvas.getContext('2d');
		if (!ctx) return;

		chartInstance = new Chart(ctx, {
			type: 'line',
			data: {
				labels: [],
				datasets: [
					{
						label: 'Completion Rate',
						data: [],
						borderColor: 'rgb(59, 130, 246)',
						backgroundColor: 'rgba(59, 130, 246, 0.1)',
						borderWidth: 3,
						fill: true,
						tension: 0.4,
						pointBackgroundColor: 'rgb(59, 130, 246)',
						pointBorderColor: '#ffffff',
						pointBorderWidth: 2,
						pointRadius: 6,
						pointHoverRadius: 8
					}
				]
			},
			options: {
				responsive: true,
				maintainAspectRatio: false,
				plugins: {
					legend: { display: false },
					tooltip: {
						mode: 'index',
						intersect: false,
						backgroundColor: 'rgba(0, 0, 0, 0.8)',
						titleColor: '#ffffff',
						bodyColor: '#ffffff',
						borderColor: 'rgb(59, 130, 246)',
						borderWidth: 1,
						cornerRadius: 8,
						displayColors: false,
						callbacks: {
							label: function (context: any) {
								return `Completion: ${context.parsed.y}%`;
							}
						}
					}
				},
				scales: {
					x: {
						grid: { display: false },
						ticks: { color: '#6b7280' }
					},
					y: {
						beginAtZero: true,
						max: 100,
						grid: { color: 'rgba(0, 0, 0, 0.1)' },
						ticks: {
							color: '#6b7280',
							callback: function (value: string | number) {
								return value + '%';
							}
						}
					}
				},
				interaction: {
					intersect: false,
					mode: 'index'
				},
				animation: {
					duration: 1000,
					easing: 'easeInOutQuart'
				}
			}
		});

		updateChart();
	}

	function updateChart() {
		if (!chartInstance || $habits.length === 0) return;

		const data = getChartData();
		chartInstance.data.labels = data.labels;
		chartInstance.data.datasets[0].data = data.values;
		chartInstance.update('active');
	}

	function getChartData(): { labels: string[]; values: number[] } {
		const days = viewMode === 'weekly' ? 7 : 30;
		const labels: string[] = [];
		const values: number[] = [];

		for (let i = days - 1; i >= 0; i--) {
			const date = new Date();
			date.setDate(date.getDate() - i);
			const dateStr = date.toISOString().split('T')[0];

			labels.push(
				viewMode === 'weekly'
					? date.toLocaleDateString('en-US', { weekday: 'short' })
					: date.toLocaleDateString('en-US', { day: 'numeric', month: 'short' })
			);

			const total = $habits.filter((h: Habit) => new Date(h.createdAt) <= date).length;
			const completed = $habits.filter(
				(h: Habit) => h.completions[dateStr] && new Date(h.createdAt) <= date
			).length;

			const rate = total > 0 ? Math.round((completed / total) * 100) : 0;
			values.push(rate);
		}

		return { labels, values };
	}

	function getWeeklyStats() {
		const now = new Date();
		const weekStart = new Date(now);
		weekStart.setDate(now.getDate() - 6);

		let total = 0;
		let completed = 0;

		for (let i = 0; i < 7; i++) {
			const date = new Date(weekStart);
			date.setDate(weekStart.getDate() + i);
			const dateStr = date.toISOString().split('T')[0];

			const active = $habits.filter((h: Habit) => new Date(h.createdAt) <= date);
			const done = active.filter((h: Habit) => h.completions[dateStr]);

			total += active.length;
			completed += done.length;
		}

		return {
			weeklyRate: total > 0 ? Math.round((completed / total) * 100) : 0,
			completed,
			total
		};
	}

	function getBestStreak(): number {
		let maxStreak = 0;

		$habits.forEach((habit: Habit) => {
			let current = 0;
			let best = 0;
			let date = new Date();

			for (let i = 0; i < 365; i++) {
				const dateStr = date.toISOString().split('T')[0];
				if (habit.completions[dateStr] && new Date(habit.createdAt) <= date) {
					current++;
					best = Math.max(best, current);
				} else {
					current = 0;
				}
				date.setDate(date.getDate() - 1);
			}
			maxStreak = Math.max(maxStreak, best);
		});

		return maxStreak;
	}

	$: weeklyStats = getWeeklyStats();
	$: bestStreak = getBestStreak();
</script>

<div class="rounded-xl bg-white p-6 shadow-lg dark:bg-gray-800 dark:text-gray-200">
	<div class="mb-6 flex items-center justify-between">
		<h2 class="flex items-center gap-2 text-xl font-bold text-gray-800 dark:text-gray-200">
			<BarChart3 class="text-purple-500 dark:text-purple-400" size={24} />
			Progress & Statistics
		</h2>

		<!-- View Mode Toggle -->
		<div class="flex rounded-lg bg-gray-100 p-1 dark:bg-gray-700">
			<button
				class="rounded px-3 py-1 text-sm font-medium transition-all duration-200 {viewMode ===
				'weekly'
					? 'bg-white text-blue-600 shadow-sm dark:bg-gray-800 dark:text-blue-400'
					: 'text-gray-600 hover:text-gray-800 dark:text-gray-300 dark:hover:text-gray-200'}"
				on:click={() => {
					viewMode = 'weekly';
					updateChart();
				}}
			>
				7 Days
			</button>
			<button
				class="rounded px-3 py-1 text-sm font-medium transition-all duration-200 {viewMode ===
				'monthly'
					? 'bg-white text-blue-600 shadow-sm dark:bg-gray-800 dark:text-blue-400'
					: 'text-gray-600 hover:text-gray-800 dark:text-gray-300 dark:hover:text-gray-200'}"
				on:click={() => {
					viewMode = 'monthly';
					updateChart();
				}}
			>
				30 Days
			</button>
		</div>
	</div>

	{#if $habits.length === 0}
		<div class="py-12 text-center">
			<div class="mb-4 text-6xl">📊</div>
			<h3 class="mb-2 text-lg font-semibold text-gray-700 dark:text-gray-200">No Data Available</h3>
			<p class="text-gray-500 dark:text-gray-400">Add habits and start tracking to see progress!</p>
		</div>
	{:else}
		<!-- Stats Cards -->
		<div class="mb-6 grid grid-cols-1 gap-4 md:grid-cols-3">
			<div
				class="rounded-lg bg-gradient-to-r from-green-400 to-green-600 p-4 text-white dark:from-green-500 dark:to-green-700"
			>
				<div class="flex items-center justify-between">
					<div>
						<p class="text-sm text-green-100 dark:text-green-200">This Week</p>
						<p class="text-2xl font-bold">{weeklyStats.weeklyRate}%</p>
						<p class="text-xs text-green-100 dark:text-green-200">
							{weeklyStats.completed}/{weeklyStats.total} completed
						</p>
					</div>
					<TrendingUp size={32} class="text-green-200 dark:text-green-300" />
				</div>
			</div>

			<div
				class="rounded-lg bg-gradient-to-r from-blue-400 to-blue-600 p-4 text-white dark:from-blue-500 dark:to-blue-700"
			>
				<div class="flex items-center justify-between">
					<div>
						<p class="text-sm text-blue-100 dark:text-blue-200">Total Habits</p>
						<p class="text-2xl font-bold">{$habits.length}</p>
						<p class="text-xs text-blue-100 dark:text-blue-200">active habits</p>
					</div>
					<Calendar size={32} class="text-blue-200 dark:text-blue-300" />
				</div>
			</div>

			<div
				class="rounded-lg bg-gradient-to-r from-orange-400 to-orange-600 p-4 text-white dark:from-orange-500 dark:to-orange-700"
			>
				<div class="flex items-center justify-between">
					<div>
						<p class="text-sm text-orange-100 dark:text-orange-200">Best Streak</p>
						<p class="text-2xl font-bold">{bestStreak}</p>
						<p class="text-xs text-orange-100 dark:text-orange-200">days in a row</p>
					</div>
					<div class="text-2xl text-orange-200 dark:text-orange-300">🔥</div>
				</div>
			</div>
		</div>

		<!-- Chart -->
		<div class="rounded-lg bg-gray-50 p-4 dark:bg-gray-700">
			<h3 class="mb-4 text-lg font-semibold text-gray-700 dark:text-gray-200">
				Completion Rate - {viewMode === 'weekly' ? 'Last 7 Days' : 'Last 30 Days'}
			</h3>
			<div style="position: relative; height: 300px;">
				<canvas bind:this={chartCanvas}></canvas>
			</div>
		</div>

		<!-- Habit Breakdown -->
		<div class="mt-6">
			<h3 class="mb-4 text-lg font-semibold text-gray-700 dark:text-gray-200">Habit Breakdown</h3>
			<div class="space-y-3">
				{#each $habits as habit}
					{@const completions = Object.keys(habit.completions).length}
					{@const daysSince =
						Math.ceil(
							(new Date().getTime() - new Date(habit.createdAt).getTime()) / (1000 * 60 * 60 * 24)
						) + 1}
					{@const rate = (() => {
						const createdDate = new Date(habit.createdAt);
						const today = new Date();
						const totalDays =
							Math.ceil((today.getTime() - createdDate.getTime()) / (1000 * 60 * 60 * 24)) + 1;

						let validCompletions = 0;
						for (let i = 0; i < totalDays; i++) {
							const checkDate = new Date(createdDate);
							checkDate.setDate(createdDate.getDate() + i);
							const dateStr = checkDate.toISOString().split('T')[0];

							if (habit.completions[dateStr] && checkDate <= today) {
								validCompletions++;
							}
						}

						return Math.round((validCompletions / totalDays) * 100);
					})()}

					<div
						class="flex items-center justify-between rounded-lg bg-gray-50 p-3 transition-all duration-200 hover:bg-gray-100 dark:bg-gray-800 dark:hover:bg-gray-700"
					>
						<div class="flex-1">
							<p class="font-medium text-gray-800 dark:text-gray-200">{habit.name}</p>
							<div class="mt-2 h-2 w-full rounded-full bg-gray-200 dark:bg-gray-600">
								<div
									class="h-2 rounded-full bg-gradient-to-r from-blue-400 to-purple-500 transition-all duration-500 dark:from-blue-500 dark:to-purple-600"
									style="width: {rate}%"
								></div>
							</div>
						</div>
						<div class="ml-4 text-right">
							<p class="text-lg font-bold text-gray-800 dark:text-gray-200">{rate}%</p>
							<p class="text-xs text-gray-500 dark:text-gray-400">
								{Object.keys(habit.completions).filter((date) => {
									const checkDate = new Date(date);
									const createdDate = new Date(habit.createdAt);
									const today = new Date();
									return checkDate >= createdDate && checkDate <= today;
								}).length} completed in <br>{daysSince} days
							</p>
						</div>
					</div>
				{/each}
			</div>
		</div>
	{/if}
</div>
