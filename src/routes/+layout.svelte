<script lang="ts">
	import { invoke } from '@tauri-apps/api/core';
	import { emit, listen } from '@tauri-apps/api/event'
	import { seqtaFetch } from '../utils/seqtaFetch';
	import { cache } from '../utils/cache';

	import { onMount } from 'svelte';
	import '../app.css';
	import {
		Icon,
		Home,
		Newspaper,
		UserGroup,
		ClipboardDocumentList,
		BookOpen,
		Squares2x2,
		ChatBubbleLeftRight,
		DocumentText,
		AcademicCap,
		Bell,
		RectangleStack,
		ArrowLeftStartOnRectangle,
		ChartBar,
		Cog6Tooth,
		CalendarDays,
		GlobeAlt,
		ArrowRightOnRectangle,
		XMark,
		Bars3
	} from 'svelte-hero-icons';

	import { writable } from 'svelte/store';
	export const needsSetup = writable(false);

	let seqtaUrl = $state<string>('');
	let userInfo = $state<UserInfo>();
	let { children } = $props();

	let weatherEnabled = $state(false);
	let weatherLocation = $state('');
	let weatherData: any = $state(null);
	let loadingWeather = $state(false);
	let weatherError = $state('');

	async function checkSession() {
		const sessionExists = await invoke<boolean>('check_session_exists');
		needsSetup.set(!sessionExists);
		if (sessionExists) {
			loadUserInfo();
		}
	}

	onMount(checkSession);

	listen<string>('reload', (event) => {
		location.reload();
		checkSession();
	})

	async function startLogin() {
		if (!seqtaUrl) return;
		await invoke('create_login_window', { url: seqtaUrl });

		// Poll every 1.5 s until the cookie is saved (login window closes itself)
		const timer = setInterval(async () => {
			const sessionExists = await invoke<boolean>('check_session_exists');
			needsSetup.set(!sessionExists);
			if (sessionExists) clearInterval(timer);
		}, 1500);
	}

	async function getAPIData(url: string, parameters: Map<string, string>) {
		return await invoke('get_api_data', { url, parameters: Object.fromEntries(parameters) });
	}

	async function postAPIData(url: string, data: Map<string, string>) {
		return await invoke('post_api_data', { url, data: Object.fromEntries(data) });
	}

	async function handleLogout() {
		const success = await invoke('logout');
		if (success) {
			await checkSession();
		}
	}

	interface UserInfo {
		clientIP: string;
		email: string;
		id: number;
		lastAccessedTime: number;
		meta: {
			code: string;
			governmentID: string;
		};
		personUUID: string;
		saml: [{
			autologin: boolean;
			label: string;
			method: string;
			request: string;
			sigalg: URL;
			signature: string;
			slo: boolean;
			url: URL
		}];
		status: string;
		type: string;
		userCode: string;
		userDesc: string;
		userName: string;
	}

	async function loadUserInfo() {
		try {
			// Check cache first
			const cachedUserInfo = cache.get<UserInfo>('userInfo');
			if (cachedUserInfo) {
				userInfo = cachedUserInfo;
				return;
			}

			const res = await seqtaFetch('/seqta/student/login?', {
				method: 'POST',
				headers: { 'Content-Type': 'application/json; charset=utf-8' },
				body: {}
			});
			userInfo = JSON.parse(res).payload;
			
			// Cache the user info for 5 minutes
			cache.set('userInfo', userInfo);
		} catch (e) {
			console.error('Failed to load user info:', e);
		}
	}

	async function loadWeatherSettings() {
		try {
			const settings = await invoke<{ weather_enabled: boolean, weather_location: string }>('get_settings');
			weatherEnabled = settings.weather_enabled ?? false;
			weatherLocation = settings.weather_location ?? '';
		} catch (e) {
			weatherEnabled = false;
			weatherLocation = '';
		}
	}

	async function fetchWeather() {
		if (!weatherEnabled || !weatherLocation) {
			weatherData = null;
			return;
		}

		// Check cache first
		const cachedWeather = cache.get<any>('weather');
		if (cachedWeather) {
			weatherData = cachedWeather;
			return;
		}

		loadingWeather = true;
		weatherError = '';
		try {
			const geoRes = await fetch(`https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(weatherLocation)}&count=1&language=en&format=json`);
			const geoJson = await geoRes.json();
			if (!geoJson.results || !geoJson.results.length) throw new Error('Location not found');
			const { latitude, longitude, name, country } = geoJson.results[0];
			const weatherRes = await fetch(`https://api.open-meteo.com/v1/forecast?latitude=${latitude}&longitude=${longitude}&current_weather=true&timezone=auto`);
			const weatherJson = await weatherRes.json();
			weatherData = {
				...weatherJson.current_weather,
				location: name,
				country
			};

			// Cache weather data for 15 minutes
			cache.set('weather', weatherData, 15 * 60 * 1000);
		} catch (e) {
			weatherError = 'Failed to load weather.';
			weatherData = null;
		} finally {
			loadingWeather = false;
		}
	}

	onMount(async () => {
		await loadWeatherSettings();
		if (weatherEnabled && weatherLocation) fetchWeather();
	});

	$effect(() => {
		if (weatherEnabled && weatherLocation) fetchWeather();
	});

	/* Sidebar menu items */
	const menu = [
		{ label: 'Home', icon: Home, path: '/' },
		{ label: 'News', icon: Newspaper, path: '/news' },
		{ label: 'Welcome', icon: UserGroup, path: '/welcome' },
		{ label: 'Assessments', icon: ClipboardDocumentList, hasSub: true, path: '/assessments' },
		{ label: 'Courses', icon: BookOpen, hasSub: true, path: '/courses' },
		{ label: 'Dashboard', icon: Squares2x2, path: '/dashboard' },
		{ label: 'Direqt Messages', icon: ChatBubbleLeftRight, path: '/direqt-messages' },
		{ label: 'Notices', icon: Bell, path: '/notices' },
		{ label: 'Reports', icon: ChartBar, path: '/reports' },
		{ label: 'Settings', icon: Cog6Tooth, path: '/settings' },
		{ label: 'Timetable', icon: CalendarDays, path: '/timetable' }
	];

	// Mobile navigation items (top 5 most important items)
	const mobileMenu = menu.slice(0, 5);

	let isSidebarOpen = $state(false);
</script>

<div class="flex flex-col h-screen bg-slate-900">
	<!-- Mobile Header -->
	<div class="md:hidden flex items-center justify-between px-4 py-2 bg-slate-800">
		<div class="flex items-center">
			<img src="/32x32.png" alt="DesQTA Logo" class="w-8 h-8 select-none" draggable="false" />
			<span class="ml-2 text-lg font-bold tracking-wide">DesQTA</span>
		</div>
		<button 
			class="p-2 rounded-lg hover:bg-slate-700"
			on:click={() => isSidebarOpen = !isSidebarOpen}
		>
			<Icon src={isSidebarOpen ? XMark : Bars3} class="w-6 h-6" />
		</button>
	</div>

	<div class="flex flex-1 overflow-hidden">
		<!-- Sidebar (hidden on mobile unless toggled) -->
		<aside
			class="fixed md:relative inset-y-0 left-0 z-50 w-64 h-full bg-slate-900 transform transition-transform duration-300 ease-in-out md:translate-x-0"
			class:translate-x-0={isSidebarOpen}
			class:-translate-x-full={!isSidebarOpen}
		>
			<div class="flex flex-col justify-between h-full">
				<div class="flex overflow-y-auto flex-col gap-2">
					<div class="hidden md:flex items-center px-4 pt-4 pb-2">
						<img src="/32x32.png" alt="DesQTA Logo" class="mr-3 w-8 h-8 select-none" draggable="false" />
						<span class="text-lg font-bold tracking-wide">DesQTA</span>
					</div>
					{#each menu as item}
						<a 
							href={item.path} 
							class="flex items-center px-4 py-3 rounded transition-colors duration-300 hover:bg-slate-800"
							on:click={() => isSidebarOpen = false}
						>
							<Icon src={item.icon} class="mr-4 w-6 h-6" />
							<span class="text-base font-bold">{item.label}</span>
							{#if item.hasSub}
								<svg
									class="ml-auto w-4 h-4"
									fill="none"
									stroke="currentColor"
									stroke-width="2"
									viewBox="0 0 24 24"
								>
									<path stroke-linecap="round" stroke-linejoin="round" d="M9 5l7 7-7 7" />
								</svg>
							{/if}
						</a>
					{/each}
					{#if weatherEnabled && weatherLocation}
						<div class="my-4 mx-2 rounded-2xl shadow bg-gradient-to-br from-blue-900 to-blue-700 text-white p-4">
							{#if loadingWeather}
								<div>Loading weather…</div>
							{:else if weatherError}
								<div class="text-red-400">{weatherError}</div>
							{:else if weatherData}
								<div class="flex flex-col gap-1">
									<div class="text-base font-bold flex items-center gap-2">
										<span>Weather in {weatherData.location}, {weatherData.country}</span>
									</div>
									<div class="flex items-center gap-2 mt-2">
										<span class="text-2xl font-bold">{Math.round(weatherData.temperature)}°C</span>
										<span class="text-lg">{weatherData.weathercode === 0 ? '☀️' : weatherData.weathercode < 4 ? '🌤️' : weatherData.weathercode < 45 ? '☁️' : '🌧️'}</span>
										<span class="text-xs">{weatherData.windspeed} km/h wind</span>
									</div>
								</div>
							{/if}
						</div>
					{/if}
				</div>
				<div class="p-4">
					<button
						on:click={handleLogout}
						class="flex items-center w-full px-4 py-3 text-red-400 rounded transition-colors duration-300 hover:bg-slate-800"
					>
						<Icon src={ArrowLeftStartOnRectangle} class="mr-4 w-6 h-6" />
						<span class="text-base font-bold">Logout</span>
					</button>
				</div>
			</div>
		</aside>

		<!-- Main Content -->
		<main class="flex-1 overflow-y-auto">
			{#if $needsSetup}
				<div class="flex flex-col items-center justify-center h-full p-4">
					<div class="w-full max-w-md p-6 bg-slate-800 rounded-lg shadow-lg">
						<h2 class="mb-4 text-2xl font-bold">Welcome to DesQTA</h2>
						<p class="mb-4 text-slate-300">Please enter your SEQTA URL to get started.</p>
						<div class="flex gap-2">
							<input
								type="text"
								bind:value={seqtaUrl}
								placeholder="e.g. seqta.wa.edu.au"
								class="flex-1 px-4 py-2 bg-slate-700 rounded focus:outline-none focus:ring-2 focus:ring-blue-500"
							/>
							<button
								on:click={startLogin}
								class="px-4 py-2 text-white bg-blue-600 rounded hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500"
							>
								Login
							</button>
						</div>
					</div>
				</div>
			{:else}
				{@render children()}
			{/if}
		</main>
	</div>

	<!-- Mobile Bottom Navigation -->
	<div class="md:hidden fixed bottom-0 left-0 right-0 bg-slate-800 border-t border-slate-700">
		<div class="flex justify-around">
			{#each mobileMenu as item}
				<a
					href={item.path}
					class="flex flex-col items-center py-2 px-4 text-sm"
					class:text-blue-400={window.location.pathname === item.path}
				>
					<Icon src={item.icon} class="w-6 h-6" />
					<span class="mt-1">{item.label}</span>
				</a>
			{/each}
		</div>
	</div>
</div>

<style>
	/* Add padding to main content to account for bottom navigation on mobile */
	@media (max-width: 768px) {
		main {
			padding-bottom: 4rem;
		}
	}
</style> 