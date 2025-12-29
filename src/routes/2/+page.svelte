<script lang="ts">
	import SceneInspector from '$lib/components/overlays/SceneInspector.svelte';
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	let container: HTMLDivElement | undefined = $state();

	const scene = new THREE.Scene();

	const cube = new THREE.Mesh(
		new THREE.BoxGeometry(1, 1, 1),
		new THREE.MeshBasicMaterial({ color: 'red' })
	);
	scene.add(cube);
	const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 30);
	camera.position.z = 10;
	scene.add(camera);

	let renderer = new THREE.WebGLRenderer({ antialias: true });
	renderer.setSize(window.innerWidth, window.innerHeight);
	renderer.render(scene, camera);

	function animate() {
		requestAnimationFrame(animate);
		cube.rotation.x += 0.01;
		cube.rotation.y += 0.01;
		renderer.render(scene, camera);
	}

	onMount(() => {
		container?.appendChild(renderer.domElement);
		animate();
	});
</script>

<div class="relative h-screen w-full">
	<div bind:this={container} class="h-screen w-full"></div>
	<SceneInspector {scene} />
</div>
