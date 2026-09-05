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

	let renderVisible = $state({});
	let lastSceneKey = '';

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
			renderer._customDrawOrder = renderOrderfiles.map(file => {
				return String(originalFiles.indexOf(file));
			});
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

		if (files.length === 0) return;

		// 只有在场景变化时才重新初始化
		if (sceneKey !== lastSceneKey) {
			lastSceneKey = sceneKey;
			initializing = true;
			
			// 实际渲染顺序（正向）
			renderOrderfiles = [...files];
			// UI 显示顺序（反向）
			syncDisplayFromRender();
			
			renderVisible = Object.fromEntries(
				files.map(file => [file, true])
			);
			
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

	function saveFileInfo() {
		console.log({
			gameName,
			platform,
			appId,
			roleName,
			skinName
		});
	}

	async function writeConfig() {
		const draworder = [...renderOrderfiles];

		try {
			await invoke(
				'write_draworder_config',
				{
					dirPath: appState.directories.selectedDir,
					draworder
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
			<div id="title">扩展功能</div>

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
						<input type="checkbox" bind:checked={editMode} />
						<span>编辑</span>
					</label>
					<button onclick={saveFileInfo}>保存</button>
				</div>
			</div>

			<div id="subtitle">Alpha 模式：</div>
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
		</div>
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
	height: 20px;
	display: flex;
	align-items: center;
	padding-left: 0px;
	color: #fff;
	font-size: 20px;
}

#subtitle {
	padding: 1px 0 1px 10px;
	color: #fff;
	border-bottom: 1px solid #444;
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
	display: flex;
	padding: 1px 1px;
	align-items: center;
	height: 22px;
	min-height: 18px;
	margin-bottom: 2px;
	color: #fff;
	font-size: 18px;
	line-height: 21px;
}

.infoLabel {
	color: #f1f1f1;
	margin-right: 4px;
}

/* 编辑 / 保存 */
#fileInfoButtons {
	display: flex;
	align-items: center;
	gap: 10px;
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
</style>
