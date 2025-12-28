<script lang="ts">
	import * as THREE from 'three';
	import { onMount, onDestroy } from 'svelte';

	import { Button } from '$lib/components/ui/button';
	import { Card, CardContent, CardHeader, CardTitle } from '$lib/components/ui/card';
	import {
		Collapsible,
		CollapsibleContent,
		CollapsibleTrigger
	} from '$lib/components/ui/collapsible';
	import { Separator } from '$lib/components/ui/separator';
	import { Switch } from '$lib/components/ui/switch';
	import { Label } from '$lib/components/ui/label';
	import { Slider } from '$lib/components/ui/slider';
	import { Settings2, X, RefreshCw } from 'lucide-svelte';

	export let scene: THREE.Scene | null = null;

	let open = true;

	let groups: THREE.Group[] = [];
	let meshes: THREE.Mesh[] = [];
	let cameras: THREE.Camera[] = [];

	let selectedUuid: string | null = null;
	let selected: THREE.Object3D | null = null;

	function refresh() {
		groups = [];
		meshes = [];
		cameras = [];
		selected = null;

		if (!scene) return;

		scene.traverse((obj) => {
			if (obj instanceof THREE.Group) groups.push(obj);
			if (obj instanceof THREE.Mesh) meshes.push(obj);
			if (obj instanceof THREE.Camera) cameras.push(obj);
		});

		// Keep selection if possible
		if (selectedUuid) {
			const found = scene.getObjectByProperty('uuid', selectedUuid);
			selected = found ?? null;
		}
	}

	// Optional: light “watch” for scene changes (good for learning / dev)
	// If you don't like polling, remove this and call refresh() manually when you add/remove objects.
	let interval: number | null = null;

	onMount(() => {
		refresh();
		interval = window.setInterval(() => {
			// cheap heuristic: if counts differ, refresh
			// (not perfect, but useful during experimentation)
			const next = { g: 0, m: 0, c: 0 };
			scene?.traverse((o) => {
				if (o instanceof THREE.Group) next.g++;
				if (o instanceof THREE.Mesh) next.m++;
				if (o instanceof THREE.Camera) next.c++;
			});

			if (next.g !== groups.length || next.m !== meshes.length || next.c !== cameras.length)
				refresh();
		}, 800);
	});

	onDestroy(() => {
		if (interval) window.clearInterval(interval);
	});

	function pick(uuid: string) {
		selectedUuid = uuid;
		selected = scene?.getObjectByProperty('uuid', uuid) ?? null;
	}

	function setWireframe(mesh: THREE.Mesh, next: boolean) {
		const mat = mesh.material as any;
		if (!mat) return;
		// supports single material only; extend if you use arrays
		if (typeof mat.wireframe === 'boolean') mat.wireframe = next;
	}

	function num(v: unknown) {
		return typeof v === 'number' ? v : 0;
	}

	function isPerspective(cam: THREE.Camera): cam is THREE.PerspectiveCamera {
		return (cam as any).isPerspectiveCamera === true;
	}
</script>

<div class="pointer-events-auto absolute top-3 right-3 z-20">
	<Collapsible bind:open>
		<div class="flex justify-end gap-2">
			<Button
				variant="secondary"
				size="icon"
				class="bg-background/80 shadow-md backdrop-blur supports-backdrop-filter:bg-background/60"
				onclick={refresh}
				aria-label="Refresh scene listing"
			>
				<RefreshCw class="h-4 w-4" />
			</Button>

			<CollapsibleTrigger asChild>
				<Button
					variant="secondary"
					size="icon"
					class="bg-background/80 shadow-md backdrop-blur supports-backdrop-filter:bg-background/60"
					aria-label={open ? 'Collapse inspector' : 'Open inspector'}
				>
					{#if open}
						<X class="h-4 w-4" />
					{:else}
						<Settings2 class="h-4 w-4" />
					{/if}
				</Button>
			</CollapsibleTrigger>
		</div>

		<CollapsibleContent>
			<Card
				class="mt-3 w-90 bg-background/80 shadow-lg backdrop-blur supports-backdrop-filter:bg-background/60"
			>
				<CardHeader class="py-3">
					<CardTitle class="text-sm">Scene Inspector</CardTitle>
				</CardHeader>

				<CardContent class="space-y-4">
					{#if !scene}
						<div class="text-xs text-muted-foreground">No scene passed in.</div>
					{:else}
						<div class="grid grid-cols-3 gap-2 text-xs">
							<div class="rounded-md border p-2">
								<div class="text-muted-foreground">Groups</div>
								<div class="font-medium tabular-nums">{groups.length}</div>
							</div>
							<div class="rounded-md border p-2">
								<div class="text-muted-foreground">Meshes</div>
								<div class="font-medium tabular-nums">{meshes.length}</div>
							</div>
							<div class="rounded-md border p-2">
								<div class="text-muted-foreground">Cameras</div>
								<div class="font-medium tabular-nums">{cameras.length}</div>
							</div>
						</div>

						<Separator />

						<div class="space-y-2">
							<div class="text-xs font-medium text-muted-foreground">Objects</div>

							<div class="max-h-[40vh] overflow-auto rounded-md border">
								{#each [...groups, ...meshes, ...cameras] as obj (obj.uuid)}
									<button
										class="flex w-full items-center justify-between gap-3 px-3 py-2 text-left text-xs hover:bg-muted/60"
										class:bg-muted={selectedUuid === obj.uuid}
										on:click={() => pick(obj.uuid)}
									>
										<span class="truncate">
											{obj.name || obj.type}
										</span>
										<span class="text-[11px] text-muted-foreground">
											{obj.type}
										</span>
									</button>
								{/each}
							</div>
						</div>

						{#if selected}
							<Separator />

							<div class="space-y-3">
								<div class="flex items-center justify-between gap-2">
									<div class="min-w-0">
										<div class="truncate text-xs font-medium">{selected.name || selected.type}</div>
										<div class="truncate text-[11px] text-muted-foreground">{selected.uuid}</div>
									</div>
									<div class="flex items-center gap-2">
										<Label class="text-[11px]">Visible</Label>
										<Switch
											checked={selected.visible}
											onCheckedChange={(v) => (selected!.visible = v)}
										/>
									</div>
								</div>

								<div class="grid grid-cols-3 gap-2">
									<div class="space-y-1">
										<Label class="text-[11px]">Px</Label>
										<input
											class="h-9 w-full rounded-md border bg-background px-2 text-xs"
											type="number"
											step="0.1"
											value={selected.position.x}
											on:input={(e) => (selected!.position.x = +e.currentTarget.value)}
										/>
									</div>
									<div class="space-y-1">
										<Label class="text-[11px]">Py</Label>
										<input
											class="h-9 w-full rounded-md border bg-background px-2 text-xs"
											type="number"
											step="0.1"
											value={selected.position.y}
											on:input={(e) => (selected!.position.y = +e.currentTarget.value)}
										/>
									</div>
									<div class="space-y-1">
										<Label class="text-[11px]">Pz</Label>
										<input
											class="h-9 w-full rounded-md border bg-background px-2 text-xs"
											type="number"
											step="0.1"
											value={selected.position.z}
											on:input={(e) => (selected!.position.z = +e.currentTarget.value)}
										/>
									</div>
								</div>

								{#if selected instanceof THREE.Mesh}
									<div class="flex items-center justify-between">
										<Label class="text-[11px]">Wireframe</Label>
										<Switch
											checked={Boolean((selected.material as any)?.wireframe)}
											onCheckedChange={(v) => setWireframe(selected as THREE.Mesh, v)}
										/>
									</div>
								{/if}

								{#if selected instanceof THREE.Camera && isPerspective(selected as THREE.Camera)}
									<div class="space-y-2">
										<div class="flex items-center justify-between">
											<Label class="text-xs">FOV</Label>
											<span class="text-xs text-muted-foreground tabular-nums">
												{(selected as THREE.PerspectiveCamera).fov.toFixed(0)}
											</span>
										</div>
										<Slider
											value={[(selected as THREE.PerspectiveCamera).fov]}
											min={20}
											max={120}
											step={1}
											onValueChange={(v: unknown[]) => {
												(selected as THREE.PerspectiveCamera).fov = num(v[0]);
												(selected as THREE.PerspectiveCamera).updateProjectionMatrix();
											}}
										/>
									</div>
								{/if}
							</div>
						{/if}
					{/if}
				</CardContent>
			</Card>
		</CollapsibleContent>
	</Collapsible>
</div>
