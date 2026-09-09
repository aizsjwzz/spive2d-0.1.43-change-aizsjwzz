<!-- <script>
	import { appState } from '$lib/appState.svelte.js';
	import { getRenderer } from '$lib/rendererStore.svelte.js';
	import { t } from '$lib/i18n.svelte.js';
	import { saveSetting } from '$lib/settings.js'; -->

<script>
	import { appState } from '$lib/appState.svelte.js';
	import { getRenderer } from '$lib/rendererStore.svelte.js';
	import { invoke } from '@tauri-apps/api/core';
	import { showNotification } from '$lib/notificationStore.svelte.js';
	import { t } from '$lib/i18n.svelte.js';
	import { exists, readTextFile ,writeTextFile ,readFile} from '@tauri-apps/plugin-fs';
	import { join } from '@tauri-apps/api/path';

	let selectedCv = $state("zhsound");
	let soundConfig = $state({zhcv: "",jpcv: "",krcv: "",encv: ""});
	let soundList = $state([]);
	let selectedSound = $state(null);
	let currentAudio = null;
	
	let { onFileSortModeChange } = $props();

	let renderVisible = $state({});
	let lastSceneKey = '';
	let lastFilesKey = '';

	let renderOrderfiles = $state([]);      // 实际渲染顺序（正向）
	let displayOrder = $state([]);          // UI 显示顺序（反向）
	let selectedIndex = $state(0);

	let visible = $state(false);

	let gameName = $state("");
	let platform = $state("");
	let appId = $state("");
	let roleName = $state("");
	let skinName = $state("");
	let editMode = $state(false);
	let showedit = $state(false);
	let editinfoMode = $state(false);
	let previousFileInfo = $state(null);
	let currentFileHasInfo = $state(false);

	function getCurrentCv() {
		const cvKey = {
			zhsound: 'zhcv',
			jpsound: 'jpcv',
			krsound: 'krcv',
			ensound: 'encv'
		}[selectedCv];
		return soundConfig[cvKey] ?? "";
	}
	//读取音频text
	function getCurrentText(sound) {
		const textKey = {
			zhsound: 'text',
			jpsound: 'jptext',
			krsound: 'krtext',
			ensound: 'entext'
		}[selectedCv];

		return sound[textKey] ?? "";
	}

	// 从实际渲染顺序同步到 UI 显示顺序
	function syncDisplayFromRender() {
		displayOrder = [...renderOrderfiles].reverse();
	}

	// 从 UI 显示顺序同步到实际渲染顺序
	function syncRenderFromDisplay() {
		renderOrderfiles = [...displayOrder].reverse();
		applyRenderOrder();
	}

	function applyRenderOrder() {
		const renderer = getRenderer();
		if (renderer?._skeletons && renderer?._fileNames?.files) {
			const originalFiles = renderer._fileNames.files;
			const isMerged = renderer._fileNames.isMerged;

			renderer._customDrawOrder = renderOrderfiles
				.map(file => {
					const index = originalFiles.indexOf(file);
					if (index < 0) return null;

					return String(isMerged ? index : index + 1);
				})
				.filter(key => key !== null)
				.filter(key => renderer._skeletons[key]);

			renderer.render(0);
		}
	}

	// 使用 $effect.pre 在 DOM 更新前执行，避免循环
	let initializing = false;
	
	$effect.pre(() => {
		const sceneKey =
			`${appState.directories.selectedDir}_${appState.directories.selectedScene}`;

		const files =
			appState.directories.files?.[appState.directories.selectedDir]?.[
				appState.directories.selectedScene
			]?.files ?? [];

		const filesKey = files.join('|');

		if (
			sceneKey !== lastSceneKey ||
			filesKey !== lastFilesKey
		) {
			lastSceneKey = sceneKey;
			lastFilesKey = filesKey;
			initializing = true;

			// 实际渲染顺序（正向）
			renderOrderfiles = [...files];

			// UI 显示顺序（反向）
			syncDisplayFromRender();

			renderVisible = Object.fromEntries(
				files.map(file => [file, true])
			);

			selectedIndex = 0;

			initializing = false;
		}
	});

	// 单独处理 renderVisible 的初始化
	$effect.pre(() => {
		if (initializing) return;
		for (const file of renderOrderfiles) {
			if (renderVisible[file] === undefined) {
				renderVisible[file] = true;
			}
		}
	});

	function toggle() {
		visible = !visible;
	}

	function selectFile(index) {
		selectedIndex = index;
	}

	function moveUp() {
		if (selectedIndex <= 0) return;

		[
			displayOrder[selectedIndex],
			displayOrder[selectedIndex - 1]
		] = [
			displayOrder[selectedIndex - 1],
			displayOrder[selectedIndex]
		];

		selectedIndex--;

		syncRenderFromDisplay();
	}

	function moveDown() {
		if (selectedIndex >= displayOrder.length - 1) return;

		[
			displayOrder[selectedIndex],
			displayOrder[selectedIndex + 1]
		] = [
			displayOrder[selectedIndex + 1],
			displayOrder[selectedIndex]
		];

		selectedIndex++;

		syncRenderFromDisplay();
	}
	function clearFileInfo() {
		gameName = "";
		platform = "";
		appId = "";
		roleName = "";
		skinName = "";
	}

	function applyPreviousFileInfo() {
		if (!previousFileInfo) {
			clearFileInfo();
			return;
		}

		gameName = previousFileInfo.gameName;
		platform = previousFileInfo.platform;
		appId = previousFileInfo.appId;
		roleName = previousFileInfo.roleName;
		skinName = previousFileInfo.skinName;
	}
	//播放音频
	async function playSound(sound) {
		if (currentAudio) {
			currentAudio.pause();
			currentAudio.currentTime = 0;
			currentAudio = null;
		}

		selectedSound = sound;

		const soundPath = sound[selectedCv];
		if (!soundPath) return;

		try {
			const soundsPath = await join(
				appState.directories.selectedDir,
				soundPath
			);

			const data = await readFile(soundsPath);
			const blob = new Blob([data], { type: 'audio/wav' });
			const url = URL.createObjectURL(blob);
			const audio = new Audio(url);

			currentAudio = audio;

			audio.onended = () => {
				if (currentAudio === audio) {
					currentAudio = null;
					selectedSound = null;
				}

				URL.revokeObjectURL(url);
			};

			await audio.play();
		} catch (error) {
			console.error('[AUDIO DEBUG] Failed to play sound:', error);
		}
	}
	//读取info
	async function readFileInfo(dirPath) {
		if (!dirPath) return;

		currentFileHasInfo = false;

		try {
			const configPath = await join(dirPath, 'spive2d_config.json');

			if (!(await exists(configPath))) {
				if (editinfoMode) {
					applyPreviousFileInfo();
				} else {
					clearFileInfo();
				}
				return;
			}

			const configText = await readTextFile(configPath);
			const config = JSON.parse(configText);
			const info = config.info;
			const sound = config.sound ?? {};
			soundConfig = {
				zhcv: sound.zhcv ?? "",
				jpcv: sound.jpcv ?? "",
				krcv: sound.krcv ?? "",
				encv: sound.encv ?? ""
			};
			soundList = Object.entries(sound)
				.filter(([key, value]) => {
					return !['zhcv', 'jpcv', 'krcv', 'encv'].includes(key)
						&& value
						&& typeof value === 'object';
				})
				.map(([name, value]) => ({
					name,
					text: value.text ?? "",
					jptext: value.jptext ?? "",
					krtext: value.krtext ?? "",
					entext: value.entext ?? "",
					zhsound: value.zhsound ?? "",
					jpsound: value.jpsound ?? "",
					krsound: value.krsound ?? "",
					ensound: value.ensound ?? ""
				}));

			if (!info) {
				if (editinfoMode) {
					applyPreviousFileInfo();
				} else {
					clearFileInfo();
				}
				return;
			}

			await writeTextFile(
				configPath,
				JSON.stringify(config, null, 4)
			);

			currentFileHasInfo = true;

			previousFileInfo = {
				gameName: info.gameName ?? "",
				platform: info.platform ?? "",
				appId: info.appId ?? "",
				roleName: info.roleName ?? "",
				skinName: info.skinName ?? ""
			};

			gameName = previousFileInfo.gameName;
			platform = previousFileInfo.platform;
			appId = previousFileInfo.appId;
			roleName = previousFileInfo.roleName;
			skinName = previousFileInfo.skinName;

		} catch (error) {
			console.warn('[CONFIG DEBUG] Failed to read file info:', error);

			currentFileHasInfo = false;

			if (editinfoMode) {
				applyPreviousFileInfo();
			} else {
				clearFileInfo();
			}
		}
	}

	$effect(() => {
    const dirPath = appState.directories.selectedDir;

		if (dirPath) {
			readFileInfo(dirPath);
		}
	});

	$effect(() => {
		if (!editinfoMode || currentFileHasInfo) return;

		if (previousFileInfo) {
			applyPreviousFileInfo();
		} else {
			clearFileInfo();
		}
	});

	async function saveFileInfo() {
		const dirPath = appState.directories.selectedDir;

		if (!dirPath) {
			showNotification('未选择目录');
			return;
		}

		try {
			const configPath = await join(dirPath, 'spive2d_config.json');
			let config = {};

			if (await exists(configPath)) {
				const configText = await readTextFile(configPath);
				config = JSON.parse(configText);
			}

			config.info = {
				appId,
				gameName,
				platform,
				roleName,
				skinName
			};

			await writeTextFile(
				configPath,
				JSON.stringify(config, null, 4)
			);

			showNotification('文件信息保存成功');
		} catch (error) {
			console.error('[CONFIG DEBUG] Failed to save file info:', error);
			showNotification('文件信息保存失败');
		}
	}

	//Alpha 模式控件

	let alphaMode = $state(appState.alphaMode);

	$effect(() => {
		alphaMode = appState.alphaMode;
	});

	async function handleAlphaModeChange(e) {
		alphaMode = e.target.value;
		appState.alphaMode = alphaMode;

		const renderer = getRenderer();
		if (renderer && renderer.setAlphaMode) {
			await renderer.setAlphaMode(alphaMode);
		}
	}

	async function writeConfig() {
		const draworder = [...displayOrder];

		try {
			await invoke(
				'write_draworder_config',
				{
					dirPath: appState.directories.selectedDir,
					draworder,
					alphaMode
				}
			);
			showNotification('配置写入成功');
		} catch (error) {
			showNotification('配置写入失败');
		}
	}
</script>

<!-- HTML 部分保持不变 -->
<div id="rightToolbar">
	<button id="toggleBtn" onclick={toggle}>
		{visible ? '>' : '<'}
	</button>

	{#if visible}
		<div id="panel">
			<div id="title">Plugin: v1.4</div>

			<div id="subtitle">文件信息:</div>

			<div id="fileInfo">
				<div class="infoGameName">
					{#if editMode}
						<input type="text" bind:value={gameName} />
					{:else}
						<span>{gameName || "无数据"}</span>
					{/if}
				</div>

				<div class="infoSub">
					{#if editMode}
						<input type="text" bind:value={platform} placeholder="平台" />
						<input type="text" bind:value={appId} placeholder="包名" />
					{:else}
						<span>{platform || "无数据"}</span>
						<span>{appId || "无数据"}</span>
					{/if}
				</div>

				<div class="infoItem">
					<span class="infoLabel">角色：</span>
					{#if editMode}
						<input type="text" bind:value={roleName} />
					{:else}
						<span>{roleName || "无数据"}</span>
					{/if}
				</div>

				<div class="infoItem">
					<span class="infoLabel">皮肤：</span>
					{#if editMode}
						<input type="text" bind:value={skinName} />
					{:else}
						<span>{skinName || "无数据"}</span>
					{/if}
				</div>

				<div id="fileInfoButtons">
					<label class="editCheck">
						<input type="checkbox" bind:checked={showedit} />
						<span>主页显示</span>
					</label>
					<label class="editCheck">
						<input
							type="checkbox"
							bind:checked={editMode}
							onchange={() => {
								if (!editMode) {
									editinfoMode = false;
								}
							}}
						/>
						<span>编辑</span>
					</label>
					<label class="editCheck">
						<input type="checkbox" bind:checked={editinfoMode} disabled={!editMode}/>
						<span>继承</span>
					</label>
					<button onclick={saveFileInfo}>保存</button>
				</div>
			</div>

			<div class="alphaModeRow">
				<div id="subtitle">Alpha 模式：</div>
				<select id="alphaModeSelector" value={alphaMode} onchange={handleAlphaModeChange}>
					<option value="pma">{t('alphaModePMA')}</option>
					<option value="unpack">{t('alphaModeUnpack')}</option>
					<option value="npm">{t('alphaModeNPM')}</option>
				</select>
			</div>

			<div class="sortModeRow">
				<div id="subtitle">排序方式：</div>
				<select
					id="sortModeSelector"
					bind:value={appState.fileSortMode}
					onchange={() => {
						console.log('[SORT DEBUG] RightToolbar change:', appState.fileSortMode);
						onFileSortModeChange();
					}}
				>
					<option value="asc">{t('asc')}</option>
					<option value="desc">{t('desc')}</option>
					<option value="config">{t('config')}</option>
				</select>
			</div>
			
			<div id="subtitle">渲染顺序控制台:</div>

			<div id="renderOrderfileList">
				{#each displayOrder as file, index}
					<!-- svelte-ignore a11y_no_static_element_interactions -->
					<!-- svelte-ignore a11y_click_events_have_key_events -->
					<div
						class:selected={index === selectedIndex}
						class="renderOrderfileItem"
						onclick={() => selectFile(index)}
					>
						<input
							class="renderVisibleCheck"
							type="checkbox"
							checked={renderVisible[file]}
							onchange={() => {
								renderVisible[file] = !renderVisible[file];
								const renderer = getRenderer();
								if (renderer) {
									renderer._hiddenFiles = {};
									for (const name of Object.keys(renderVisible)) {
										if (renderVisible[name] === false) {
											renderer._hiddenFiles[name] = true;
										}
									}
								}
							}}
							onclick={(e) => e.stopPropagation()}
						/>
						{file}
					</div>
				{/each}
			</div>

			<div id="renderOrderbuttons">
				<button onclick={moveUp}>上移</button>
				<button onclick={moveDown}>下移</button>
				<button onclick={writeConfig}>写入配置</button>
			</div>

			<div id="audioCvTitle">
					音频控制台: cv:{getCurrentCv()}
				</div>

			<div id="audioCvButtons">
				{#if soundConfig.zhcv}
					<button
						class:active={selectedCv === "zhsound"}
						onclick={() => selectedCv = "zhsound"}
					>
						中文
					</button>
				{/if}

				{#if soundConfig.jpcv}
					<button
						class:active={selectedCv === "jpsound"}
						onclick={() => selectedCv = "jpsound"}
					>
						日语
					</button>
				{/if}

				{#if soundConfig.krcv}
					<button
						class:active={selectedCv === "krsound"}
						onclick={() => selectedCv = "krsound"}
					>
						韩语
					</button>
				{/if}

				{#if soundConfig.encv}
					<button
						class:active={selectedCv === "ensound"}
						onclick={() => selectedCv = "ensound"}
					>
						英语
					</button>
				{/if}
			</div>

			{#each soundList as sound}
				<div class="audioItem">
					<button onclick={() => playSound(sound)}>
						♫ {sound.name}
					</button>
					<span>{sound[selectedCv]?.split('/').pop() || ""}</span>
				</div>
			{/each}
			

		</div>
	{/if}
</div>
//显示信息
{#if showedit}
    <div id="infoText">
        {gameName}【{roleName}：{skinName}】
    </div>
{/if}

<div id="soundInfoText">
	{#if selectedSound}
		"{selectedSound[
			{
				zhsound: "text",
				jpsound: "jptext",
				krsound: "krtext",
				ensound: "entext"
			}[selectedCv]
		] || selectedSound.text || ""}"
	{/if}
</div>

<style>
#rightToolbar {
	position: fixed;
	top: 0;
	right: 0;
	height: 100vh;
	z-index: 100;
}

#toggleBtn {
	position: absolute;
	left: -15px;
	top: 50%;
	transform: translateY(-50%);
	width: 15px;
	height: 60px;
	background: var(--sidebar-color);
	border: var(--border-color);
	border-radius: 5px 0 0 6px;
	color: #ccc;
	cursor: pointer;
	display: flex;
	align-items: center;
	justify-content: center;
	line-height: 1;
}

#panel {
	width: 260px;
	height: 100vh;
	padding: 5px;
	background: var(--sidebar-color);
	border-left: 1px solid #444;
	z-index: 100;
	display: flex;
	flex-direction: column;
}

#title {
	height: 15px;
	padding: 0px 1px 5px 5px;
	display: flex;
	align-items: center;
	padding-left: 0px;
	color: #fff;
	font-size: 20px;
}

#subtitle {
	padding: 1px 0 1px 5px;
	color: #fff;
	border-bottom: 1px solid #444;
}

#infoText {
  position: fixed;
  bottom: 5px;
  left: 50%;
  transform: translateX(-50%);
  color: white;
  font-size: 25px;
  z-index: 100;
  text-shadow:
    -1px -1px 0 #000,
     1px -1px 0 #000,
    -1px  1px 0 #000,
     1px  1px 0 #000;
}

#soundInfoText {
  position: fixed;
  bottom: 45px;
  text-align: center;
  line-height: 1.1;
  left: 50%;
  transform: translateX(-50%);
  color: white;
  font-size: 20px;
  z-index: 100;
  text-shadow:
    -1px -1px 0 #000,
     1px -1px 0 #000,
    -1px  1px 0 #000,
     1px  1px 0 #000;
}

.alphaModeRow,
.sortModeRow {
	display: grid;
	/* grid-template-columns: max-content minmax(0, 1fr); */
	grid-template-columns: 90px 1fr;
	align-items: center;
	column-gap: 0px;
	width: 100%;
	height: 25px;
	white-space: nowrap;
}

.alphaModeRow #subtitle {
	display: block;
	white-space: nowrap;
	color: var(--text-color);
	font-size: 16px;
}

#alphaModeSelector,
#sortModeSelector {
	padding-left: 2px;
	text-indent: 0;
	text-align: left;
	display: block;
	width: 100%;
	min-width: 0;
	height: 25px;
	line-height: 10px;
	border-radius: 6px;
	outline: none;
	color: var(--text-color);
	border: var(--border-color);
	font-size: 14px;
	background-color: var(--sidebar-color);
}

/* 文件信息 */
#fileInfo {
	padding: 5px 5px 2px 10px;
	margin: 0;
}

/* 游戏名称 */
.infoGameName {
	color: #fff;
	font-size: 24px;
	margin-bottom: 0px;
	line-height: 1.1;
		text-shadow: 
		-1px -1px 0 #000,
		1px -1px 0 #000,
		-1px 1px 0 #000,
		1px 1px 0 #000;

}

/* 平台 / 包名 */
.infoSub {
	display: flex;
	gap: 12px;
	color: #fff;
	font-size: 11px;
	line-height: 1;
	margin-top: 0px;
	margin-bottom: 0px;
}

/* 角色 / 皮肤 */
.infoItem {
	margin-top: 2px;
	display: flex;
	padding: 1px 1px;
	align-items: center;
	height: 22px;
	min-height: 18px;
	margin-bottom: 2px;
	color: #fff;
	font-size: 18px;
	line-height: 21px;
		line-height: 1.1;
		text-shadow: 
		-1px -1px 0 #000,
		1px -1px 0 #000,
		-1px 1px 0 #000,
		1px 1px 0 #000;
}

.infoLabel {
	color: #f1f1f1;
	margin-right: 4px;
	
}

/* 编辑 / 保存 */
#fileInfoButtons {
	display: flex;
	align-items: center;
	gap: 2px;
	margin-top: 0px;
	height: 25px;
}

.editCheck {
	display: flex;
	align-items: center;
	gap: 4px;
	color: #ccc;
	font-size: 13px;
	cursor: pointer;
}

#fileInfoButtons button {
	padding: 2px 20px;
	font-size: 13px;
	background: var(--sidebar-color);
	border: var(--border-color);
	border-radius: 4px;
	color: #ccc;
	cursor: pointer;
}

#fileInfoButtons button:hover {
	background: #555;
}

/* 编辑输入框 */
#fileInfo input {
	box-sizing: border-box;
	min-width: 0;
	padding: 1px 1px;
	background: #222;
	border: 1px solid #555;
	border-radius: 3px;
	color: #fff;
	outline: none;
}

.infoGameName input {
	width: 100%;
	font-size: 20px;
}

.infoSub input {
	width: 50%;
	font-size: 11px;
	color: #fff;
}

.infoItem input {
	flex: 1;
	font-size: 15px;
}

#fileInfo input:focus {
	border-color: #777;
}

/* 渲染顺序 */
#renderOrderfileList {
	padding-top: 4px;
}

.renderVisibleCheck {
	width: 15.5px;
	height: 15.5px;
	margin-right: 4px;
}

.renderOrderfileItem {
	padding: 5px 5px;
	color: #ccc;
	cursor: pointer;
	display: flex;
	align-items: center;
	font-size: 14px;
	line-height: 1.2;
	background: rgba(255, 255, 255, 0.06);
	border-radius: 6px;
	border: 1px solid rgba(255, 255, 255, 0.1);
}

.renderOrderfileItem:hover {
	background: #333;
}

.renderOrderfileItem.selected {
	background: #444;
	color: #fff;
}

#renderOrderbuttons {
	display: flex;
	gap: 2px;
	padding: 2px 0px;
	width: 100%;
}

#renderOrderbuttons button:hover {
	background-color: #555;
}

#renderOrderbuttons button {
	flex: 1;
	padding: 3px 2px;
	font-size: 15px;
	background: var(--sidebar-color);
	border: var(--border-color);
	border-radius: 4px;
	color: #ccc;
	cursor: pointer;
	transition: background 0.2s;
	white-space: nowrap;
}
/* 音频控制台样式 */
#audioCvButtons,
.audioItem {
    display: flex;
    align-items: center;
    gap: 2px;
    padding: 0px 0px;
    width: 100%;
}

#audioCvButtons button{
    padding: 2px 12px;
    font-size: 15px;
    background: var(--sidebar-color);
    border: var(--border-color);
    border-radius: 4px;
    color: #fff;
    cursor: pointer;
    transition: background 0.2s;
    white-space: nowrap;
}


.audioItem button {
    padding: 3px 8px;
    font-size: 15px;
    background: var(--sidebar-color);
    border: var(--border-color);
    border-radius: 20px;
    color: #fff;
    cursor: pointer;
    transition: background 0.2s;
    white-space: nowrap;
}

#audioCvButtons button:hover,
.audioItem button:hover {
    background-color: #555;
}

.audioItem span {
    color: #ccc;
    font-size: 14px;
}

</style>
