<script>
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
import mountainUrl from '@/gameassets/themountain.gltf';
import { useConfigStore } from '@/stores/config';
import { mapState } from 'pinia';

export default {
	data() {
		return {
			scrollY: 0,
			mouseX: 0,
			mouseY: 0,
			targetMouseX: 0,
			targetMouseY: 0,
		};
	},
	computed: {
		...mapState(useConfigStore, ['dark_mode']),
	},
	watch: {
		dark_mode() {
			this.updateColors();
		},
	},
	mounted() {
		this.initScene();
	},
	beforeUnmount() {
		this.cleanup();
	},
	methods: {
		initScene() {
			const container = this.$refs.container;
			if (!container) return;

			const width = window.innerWidth;
			const height = window.innerHeight;

			this.scene = new THREE.Scene();

			this.camera = new THREE.PerspectiveCamera(55, width / height, 1, 10000);
			this.camera.position.set(0, 160, 520);

			this.renderer = new THREE.WebGLRenderer({
				antialias: true,
				alpha: true,
				powerPreference: 'high-performance',
			});
			this.renderer.outputColorSpace = THREE.SRGBColorSpace;
			this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
			this.renderer.setSize(width, height);
			this.renderer.domElement.className = 'mountain-canvas';
			container.appendChild(this.renderer.domElement);

			this.sunLight = new THREE.DirectionalLight(0xffffff, 1.4);
			this.sunLight.position.set(300, 500, 300);
			this.scene.add(this.sunLight);

			this.fillLight = new THREE.DirectionalLight(0x88b0d8, 0.7);
			this.fillLight.position.set(-300, 200, -300);
			this.scene.add(this.fillLight);

			this.ambientLight = new THREE.AmbientLight(0xffffff, 0.8);
			this.scene.add(this.ambientLight);

			this.updateColors();

			this.clock = new THREE.Clock();

			const loader = new GLTFLoader();
			loader.load(mountainUrl, (gltf) => {
				const materialCache = new Map();

				gltf.scene.traverse((child) => {
					if (child.isMesh) {
						const node = child.userData?.node;
						const staticNode =
							node?.levelNodeStatic || node?.levelNodeCrumbling;
						let r = 0.5;
						let g = 0.5;
						let b = 0.5;
						let roughness = 0.85;
						let metalness = 0.05;

						if (staticNode) {
							const matId = staticNode.material ?? 0;
							const col = staticNode.color1;
							if (col) {
								r = col.r ?? 0;
								g = col.g ?? 0;
								b = col.b ?? 0;
							} else {
								if (matId === 4) {
									r = 0.52;
									g = 0.35;
									b = 0.22;
								} else if (matId === 2) {
									r = 0.65;
									g = 0.88;
									b = 0.98;
									roughness = 0.2;
								} else if (matId === 3) {
									r = 1.0;
									g = 0.28;
									b = 0.05;
								} else if (matId === 5) {
									r = 0.28;
									g = 0.82;
									b = 0.35;
								}
							}
						}

						const key = `${r.toFixed(3)}_${g.toFixed(3)}_${b.toFixed(3)}_${roughness}_${metalness}`;
						if (!materialCache.has(key)) {
							materialCache.set(
								key,
								new THREE.MeshStandardMaterial({
									color: new THREE.Color(r, g, b),
									roughness,
									metalness,
								}),
							);
						}
						child.material = materialCache.get(key);
					}
				});

				const pivot = new THREE.Group();
				gltf.scene.position.set(-2.9, -160, 33.7);
				pivot.add(gltf.scene);
				this.scene.add(pivot);
				this.pivot = pivot;
				this.model = gltf.scene;
				this.materialCache = materialCache;
			});

			this.onResize = () => {
				const w = window.innerWidth;
				const h = window.innerHeight;
				if (this.camera && this.renderer) {
					this.camera.aspect = w / h;
					this.camera.updateProjectionMatrix();
					this.renderer.setSize(w, h);
				}
			};

			this.onScroll = () => {
				this.scrollY = window.scrollY;
			};

			this.onMouseMove = (e) => {
				this.targetMouseX = (e.clientX / window.innerWidth) * 2 - 1;
				this.targetMouseY = (e.clientY / window.innerHeight) * 2 - 1;
			};

			window.addEventListener('resize', this.onResize);
			window.addEventListener('scroll', this.onScroll, { passive: true });
			window.addEventListener('mousemove', this.onMouseMove, { passive: true });

			this.scrollY = window.scrollY;

			this.animate = () => {
				this.animationFrameId = requestAnimationFrame(this.animate);

				const delta = this.clock ? this.clock.getDelta() : 0.016;
				if (this.pivot) {
					this.pivot.rotation.y += delta * 0.2;
				}

				this.mouseX += (this.targetMouseX - this.mouseX) * 0.05;
				this.mouseY += (this.targetMouseY - this.mouseY) * 0.05;

				const docHeight =
					document.documentElement.scrollHeight - window.innerHeight || 1;
				const scrollProgress = Math.min(
					Math.max(this.scrollY / docHeight, 0),
					1,
				);

				const mouseAngleX = this.mouseX * 0.2;
				const mouseAngleY = this.mouseY * 0.15;

				const radius = 550 - scrollProgress * 120;
				const targetY = 120 + scrollProgress * 240;
				const camY = 160 + scrollProgress * 260 + mouseAngleY * 60;

				this.camera.position.x = Math.sin(mouseAngleX) * radius;
				this.camera.position.z = Math.cos(mouseAngleX) * radius;
				this.camera.position.y = camY;
				this.camera.lookAt(0, targetY, 0);

				this.renderer.render(this.scene, this.camera);
			};

			this.animate();
		},
		updateColors() {
			if (!this.scene) return;
			const isDark = this.dark_mode;
			const fogColor = isDark ? 0x091421 : 0x5f8bc2;

			if (!this.scene.fog) {
				this.scene.fog = new THREE.Fog(fogColor, 250, 1400);
			} else {
				this.scene.fog.color.setHex(fogColor);
				this.scene.fog.near = 250;
				this.scene.fog.far = 1400;
			}

			if (this.sunLight) {
				this.sunLight.color.setHex(isDark ? 0xd0e0ff : 0xffffff);
				this.sunLight.intensity = isDark ? 1.0 : 1.4;
			}
			if (this.fillLight) {
				this.fillLight.color.setHex(isDark ? 0x305080 : 0x88b0d8);
				this.fillLight.intensity = isDark ? 0.5 : 0.7;
			}
			if (this.ambientLight) {
				this.ambientLight.color.setHex(isDark ? 0x404050 : 0xffffff);
				this.ambientLight.intensity = isDark ? 0.5 : 0.8;
			}
		},
		cleanup() {
			if (this.animationFrameId) {
				cancelAnimationFrame(this.animationFrameId);
			}
			window.removeEventListener('resize', this.onResize);
			window.removeEventListener('scroll', this.onScroll);
			window.removeEventListener('mousemove', this.onMouseMove);
			if (this.pivot && this.scene) {
				this.scene.remove(this.pivot);
			}

			if (this.model) {
				this.model.traverse((child) => {
					if (child.isMesh) {
						if (child.geometry) child.geometry.dispose();
					}
				});
			}
			if (this.materialCache) {
				this.materialCache.forEach((mat) => mat.dispose());
				this.materialCache.clear();
			}
			if (this.renderer) {
				this.renderer.dispose();
				if (this.renderer.domElement && this.renderer.domElement.parentNode) {
					this.renderer.domElement.parentNode.removeChild(
						this.renderer.domElement,
					);
				}
			}
		},
	},
};
</script>

<template>
	<div class="mountain-container" ref="container"></div>
</template>

<style scoped>
.mountain-container {
	position: fixed;
	top: 0;
	left: 0;
	width: 100vw;
	height: 100vh;
	z-index: 0;
	pointer-events: none;
	overflow: hidden;
}
:deep(.mountain-canvas) {
	width: 100%;
	height: 100%;
	display: block;
}
</style>
