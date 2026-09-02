<!-- <script>
	import { appState } from '$lib/appState.svelte.js';
	import { getRenderer } from '$lib/rendererStore.svelte.js';
	import { t } from '$lib/i18n.svelte.js';
	import { saveSetting } from '$lib/settings.js'; -->
<script>
	let visible = $state(false);

	let files = $state([
		"cardspine_10064_2_BG_123456",
		"cardspine_10064_2_2",
		"cardspine_10064_2_1"
	]);

	let selectedIndex = $state(0);

	function toggle() {
		visible = !visible;
	}

	function selectFile(index) {
		selectedIndex = index;
	}

	function moveUp() {
		if (selectedIndex <= 0) return;

		const temp = files[selectedIndex];
		files[selectedIndex] = files[selectedIndex - 1];
		files[selectedIndex - 1] = temp;

		selectedIndex--;
	}

	function moveDown() {
		if (selectedIndex >= files.length - 1) return;

		const temp = files[selectedIndex];
		files[selectedIndex] = files[selectedIndex + 1];
		files[selectedIndex + 1] = temp;

		selectedIndex++;
	}
</script>

<div id="rightToolbar">

	<button id="toggleBtn" onclick={toggle}>
		{visible ? '>' : '<'}
	</button>

	{#if visible}
		<div id="panel">
			<div id="title">
				文件信息
			</div>

			<div id="orderTitle">
				渲染顺序
			</div>

			<div id="fileList">
				{#each files as file, index}
					<!-- svelte-ignore a11y_no_static_element_interactions -->
					<!-- svelte-ignore a11y_click_events_have_key_events -->
					<div
						class:selected={index === selectedIndex}
						class="fileItem"
						onclick={() => selectFile(index)}
					>
						{file}
					</div>
				{/each}
			</div>

			<div id="buttons">
				<button onclick={moveUp}>↑</button>
				<button onclick={moveDown}>↓</button>
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
	width: 220px;
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
	color: #ffffff;
	font-size: 20px;
}

#orderTitle {
	padding:12px 0 6px 12px;
	color:#ccc;
	border-bottom:1px solid #444;
}

#fileList {
	padding-top:2px;
}

.fileItem {
	padding:5px 2px;
	color:#ccc;
	cursor:pointer;
  display: flex;
  align-items: center; 

  font-size: 14px;
  line-height: 1.2;

  background: rgba(255, 255, 255, 0.06);
  border-radius: 6px;
  border: 1px solid rgba(255, 255, 255, 0.1);  
}

.fileItem:hover {
	background:#333;
}

.fileItem.selected {
	background:#444;
	color:#fff;
}

#buttons {
	display:flex;
	gap:8px;
	padding:12px;
}

#buttons button {
	width:40px;
	height:30px;
}

</style>
