<script lang="ts">
	import { initDB } from '$lib/duckdb';
	import {
		MapLibre,
		NavigationControl,
		ScaleControl,
		GeoJSONSource,
		CircleLayer
	} from 'svelte-maplibre-gl';
	import type { Table, StructRowProxy } from 'apache-arrow';
	import type GeoJsonTypes from 'geojson';

	async function load_db() {
		// duckdbの初期化
		const db = await initDB();
		const conn = await db.connect();

		// spatialを有効化
		await conn.query(`
      INSTALL json;
      INSTALL spatial;
    `);
		await conn.query(`
      LOAD json;
      LOAD spatial;
    `);
		return conn;
	}

	// DB接続
	const conn_prom = load_db();

	let map: maplibregl.Map | undefined = $state.raw();

	// 読み込んだデータのGeoJSON
	let geoJsonSource: GeoJsonTypes.FeatureCollection = $state({
		type: 'FeatureCollection',
		features: []
	});
	// 結果のGeoJSON
	let resultGeoJsonSource: GeoJsonTypes.Feature = $state({
		type: 'Feature',
		properties: {},
		geometry: {
			type: 'Point',
			coordinates: [0, 0]
		}
	});

	let resultCentroidQuery = $state(new Promise(() => {}));
	let resultGeoJsonQuery = $state(new Promise(() => {}));

	let loadGeojsonFeatures: GeoJsonTypes.Feature[] = $state([]);

	// MULTIPOINTの文字列を作るための配列たち
	const pointsArrayTemp: string[] = [];
	let pointsArray: string[] = [];
	let pointsMultiString: string = $state('');

	// ローディングフラグ
	let loading = $state(false);

	// 重心のテキスト
	let centroidText = $state('');

	// ボタンクリック時にGeoJSONを読み込んで重心を計算する
	async function handleLoadGeoJSON() {
		const urlInput = document.getElementById('url') as HTMLInputElement;
		const url = urlInput.value;

		// textが空なら何もしない
		if (url === '') {
			return;
		}

		// ローディングフラグを立てる
		loading = true;

		// 初期化
		pointsArrayTemp.length = 0;
		pointsArray.length = 0;
		centroidText = '';
		loadGeojsonFeatures.length = 0;

		try {
			await initializeDatabase(url);
			await loadGeoJsonData();
			await calculateCentroid();
		} catch (e: any) {
			alert(e.message);
		} finally {
			loading = false;
		}
	}

	async function initializeDatabase(url: string) {
		const conn = await conn_prom;
		await conn.query(`
			DROP TABLE IF EXISTS temp;
			CREATE TABLE temp AS SELECT * FROM ST_Read('${url}');
		`);
	}

	async function loadGeoJsonData() {
		const conn = await conn_prom;
		const result: Table<any> = (await conn.query(`
			SELECT ST_AsGeoJSON(geom) as geom FROM temp;
		`)) as unknown as Table<any>;
		const resultRows: StructRowProxy<any>[] = result.toArray();

		for (let i = 0; i < resultRows.length; i++) {
			const geometry = JSON.parse(resultRows[i].geom);
			pointsArrayTemp.push(geometry.coordinates[0] + ' ' + geometry.coordinates[1]);

			loadGeojsonFeatures.push({
				type: 'Feature',
				properties: {},
				geometry: geometry
			});
		}

		// 読み込んだデータを地図に反映
		geoJsonSource = {
			type: 'FeatureCollection',
			features: loadGeojsonFeatures
		};

		// 重複排除
		pointsArray = Array.from(new Set(pointsArrayTemp));
		// 計算用のMULTIPOINT文字列を作成
		pointsMultiString = 'MULTIPOINT(' + pointsArray.join(',') + ')';
	}

	async function calculateCentroid() {
		const conn = await conn_prom;
		const resultCentroid: Table<any> = (await conn.query(`
			SELECT ST_AsGeoJSON(ST_Centroid(ST_GeomFromText('${pointsMultiString}'))) as geom;
		`)) as unknown as Table<any>;
		const resultCentroidRows: StructRowProxy<any>[] = resultCentroid.toArray();

		// 重心を地図に反映
		resultGeoJsonSource = {
			type: 'Feature',
			properties: {},
			geometry: JSON.parse(resultCentroidRows[0].geom)
		};

		const centroid = JSON.parse(resultCentroidRows[0].geom).coordinates;
		centroidText = centroid[0] + ', ' + centroid[1];
		map?.flyTo({ center: centroid, zoom: 8 });
	}
</script>

<MapLibre
	bind:map
	class="h-[60vh] min-h-[300px]"
	style="https://basemaps.cartocdn.com/gl/voyager-gl-style/style.json"
	zoom={3.5}
	center={{ lng: 137, lat: 36 }}
>
	<NavigationControl />
	<ScaleControl />
	<GeoJSONSource data={geoJsonSource}>
		<CircleLayer paint={{ 'circle-radius': 5, 'circle-color': '#1C64F2' }} />
	</GeoJSONSource>
	<GeoJSONSource data={resultGeoJsonSource}>
		<CircleLayer paint={{ 'circle-radius': 10, 'circle-color': '#b00000' }} />
	</GeoJSONSource>
</MapLibre>

<div class="mt-4 pl-4">
	<h1
		class="font-BizUDPGothic mb-4 text-4xl font-extrabold leading-none tracking-tight text-gray-900 md:text-5xl lg:text-6xl"
	>
		マルチポイント重心計算くん
	</h1>

	<p class="text-lg text-gray-700 font-BizUDPGothic">
		GeoJSONファイルのURLを入力し、Load GeoJSONボタンを押してください。
	</p>
	<p class="text-lg text-gray-700 font-BizUDPGothic">
		マルチポイント以外のGeoJSONファイルのURLを入れても何も起こりませんので要注意のこと。
	</p>
	<p class="mb-4 text-lg text-gray-700 font-BizUDPGothic">URLでアクセスできるGeoJSONのみ対応。（改良予定です）</p>

	<!-- URLを入力するテキストボックスと、確定するボタンを表示する -->
	<div class="w-full min-w-[800px] max-w-sm">
		<input
			type="text"
			id="url"
			class="ease w-full rounded-md border border-slate-200 bg-transparent bg-white px-3 py-2 text-sm text-slate-700 shadow-sm transition duration-300 placeholder:text-slate-400 hover:border-slate-300 focus:border-slate-400 focus:shadow focus:outline-none"
		/>
	</div>

	<button
		class="mt-4 rounded-md border border-slate-300 bg-white px-4 py-2 text-center text-sm text-slate-600 shadow-sm transition-all hover:border-slate-800 hover:bg-slate-800 hover:text-white hover:shadow-lg focus:border-slate-800 focus:bg-slate-800 focus:text-white active:border-slate-800 active:bg-slate-800 active:text-white disabled:pointer-events-none disabled:opacity-50 disabled:shadow-none"
		onclick={handleLoadGeoJSON}>Load GeoJSON</button
	>

	{#if centroidText != ''}
		<p class="mt-4 text-lg text-gray-700">重心の座標は{centroidText}です。</p>
	{/if}
</div>
<!-- 全面を覆うローディングコンテナ -->
{#if loading}
	<div
		class="fixed left-0 top-0 z-50 flex h-full w-full items-center justify-center bg-gray-900 bg-opacity-50"
	>
		<div
			class="h-16 w-16 animate-spin rounded-full border-4 border-t-[4px] border-gray-200 border-t-blue-500"
		></div>
	</div>
{/if}

<style>
	@keyframes spin {
		0% {
			transform: rotate(0deg);
		}
		100% {
			transform: rotate(360deg);
		}
	}
	.animate-spin {
		animation: spin 1s linear infinite;
	}
</style>
