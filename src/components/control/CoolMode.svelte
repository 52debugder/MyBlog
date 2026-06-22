<script lang="ts">
	import { onDestroy, onMount } from "svelte";

	export interface CoolParticleOptions {
		particle?: string;
		size?: number;
		particleCount?: number;
		speedHorz?: number;
		speedUp?: number;
	}

	type Particle = {
		element: HTMLElement | SVGSVGElement;
		left: number;
		size: number;
		top: number;
		direction: number;
		speedHorz: number;
		speedUp: number;
		spinSpeed: number;
		spinVal: number;
	};

	type CoolModeState = Window & {
		__coolModeContainer?: HTMLDivElement;
		__coolModeInstances?: number;
	};

	export let options: CoolParticleOptions | undefined = undefined;

	let root: HTMLSpanElement | null = null;
	let container: HTMLDivElement | null = null;
	let particles: Particle[] = [];
	let autoAddParticle = false;
	let mouseX = 0;
	let mouseY = 0;
	let animationFrame: number | undefined;
	let lastParticleTimestamp = 0;
	const sizes = [15, 20, 25, 35, 45];
	const limit = 45;
	const defaultParticle = "circle";
	const SVG_NS = "http://www.w3.org/2000/svg";

	function getState(): CoolModeState {
		return window as CoolModeState;
	}

	function getContainer() {
		const state = getState();

		if (state.__coolModeContainer) {
			return state.__coolModeContainer;
		}

		const existingContainer = document.getElementById("_coolMode_effect");
		if (existingContainer) {
			state.__coolModeContainer = existingContainer as HTMLDivElement;
			return state.__coolModeContainer;
		}

		const newContainer = document.createElement("div");
		newContainer.setAttribute("id", "_coolMode_effect");
		newContainer.setAttribute(
			"style",
			"overflow:hidden; position:fixed; inset:0; pointer-events:none; z-index:2147483647",
		);

		document.body.appendChild(newContainer);
		state.__coolModeContainer = newContainer;

		return newContainer;
	}

	function incrementInstances() {
		const state = getState();
		state.__coolModeInstances = (state.__coolModeInstances ?? 0) + 1;
		return state.__coolModeInstances;
	}

	function decrementInstances() {
		const state = getState();
		state.__coolModeInstances = Math.max(0, (state.__coolModeInstances ?? 1) - 1);

		if (state.__coolModeInstances === 0) {
			state.__coolModeContainer?.remove();
			state.__coolModeContainer = undefined;
		}
	}

	function appendCircleParticle(particle: HTMLDivElement, size: number) {
		const circleSVG = document.createElementNS(SVG_NS, "svg");
		const circle = document.createElementNS(SVG_NS, "circle");

		circle.setAttributeNS(null, "cx", (size / 2).toString());
		circle.setAttributeNS(null, "cy", (size / 2).toString());
		circle.setAttributeNS(null, "r", (size / 2).toString());
		circle.setAttributeNS(null, "fill", `hsl(${Math.random() * 360}, 70%, 50%)`);

		circleSVG.appendChild(circle);
		circleSVG.setAttribute("width", size.toString());
		circleSVG.setAttribute("height", size.toString());

		particle.appendChild(circleSVG);
	}

	function appendImageParticle(
		particle: HTMLDivElement,
		imageSrc: string,
		size: number,
	) {
		const image = document.createElement("img");
		image.src = imageSrc;
		image.width = size;
		image.height = size;
		image.alt = "";
		image.style.borderRadius = "50%";

		particle.appendChild(image);
	}

	function appendTextParticle(
		particle: HTMLDivElement,
		particleContent: string,
		size: number,
	) {
		const fontSizeMultiplier = 3;
		const emojiSize = size * fontSizeMultiplier;
		const content = document.createElement("div");

		content.textContent = particleContent;
		content.style.fontSize = `${emojiSize}px`;
		content.style.lineHeight = "1";
		content.style.textAlign = "center";
		content.style.width = `${size}px`;
		content.style.height = `${size}px`;
		content.style.display = "flex";
		content.style.alignItems = "center";
		content.style.justifyContent = "center";
		content.style.transform = `scale(${fontSizeMultiplier})`;
		content.style.transformOrigin = "center";

		particle.appendChild(content);
	}

	function generateParticle() {
		const particleType = options?.particle || defaultParticle;
		const size = options?.size || sizes[Math.floor(Math.random() * sizes.length)];
		const speedHorz = options?.speedHorz || Math.random() * 10;
		const speedUp = options?.speedUp || Math.random() * 25;
		const spinVal = Math.random() * 360;
		const spinSpeed = Math.random() * 35 * (Math.random() <= 0.5 ? -1 : 1);
		const top = mouseY - size / 2;
		const left = mouseX - size / 2;
		const direction = Math.random() <= 0.5 ? -1 : 1;

		const particle = document.createElement("div");

		if (particleType === "circle") {
			appendCircleParticle(particle, size);
		} else if (particleType.startsWith("http") || particleType.startsWith("/")) {
			appendImageParticle(particle, particleType, size);
		} else {
			appendTextParticle(particle, particleType, size);
		}

		particle.style.position = "absolute";
		particle.style.transform = `translate3d(${left}px, ${top}px, 0px) rotate(${spinVal}deg)`;

		container?.appendChild(particle);

		particles.push({
			direction,
			element: particle,
			left,
			size,
			speedHorz,
			speedUp,
			spinSpeed,
			spinVal,
			top,
		});
	}

	function refreshParticles() {
		particles.forEach((particle) => {
			particle.left = particle.left - particle.speedHorz * particle.direction;
			particle.top = particle.top - particle.speedUp;
			particle.speedUp = Math.min(particle.size, particle.speedUp - 1);
			particle.spinVal = particle.spinVal + particle.spinSpeed;

			if (
				particle.top >=
				Math.max(window.innerHeight, document.body.clientHeight) + particle.size
			) {
				particles = particles.filter((item) => item !== particle);
				particle.element.remove();
				return;
			}

			particle.element.setAttribute(
				"style",
				[
					"position:absolute",
					"will-change:transform",
					`top:${particle.top}px`,
					`left:${particle.left}px`,
					`transform:rotate(${particle.spinVal}deg)`,
				].join(";"),
			);
		});
	}

	function updateMousePosition(event: MouseEvent | TouchEvent) {
		if ("touches" in event) {
			mouseX = event.touches?.[0]?.clientX ?? 0;
			mouseY = event.touches?.[0]?.clientY ?? 0;
			return;
		}

		mouseX = event.clientX;
		mouseY = event.clientY;
	}

	function tapHandler(event: MouseEvent | TouchEvent) {
		updateMousePosition(event);
		autoAddParticle = true;
	}

	function disableAutoAddParticle() {
		autoAddParticle = false;
	}

	function loop() {
		const currentTime = performance.now();
		const particleGenerationDelay = 30;
		const particleCount = options?.particleCount ?? limit;

		if (
			autoAddParticle &&
			particles.length < particleCount &&
			currentTime - lastParticleTimestamp > particleGenerationDelay
		) {
			generateParticle();
			lastParticleTimestamp = currentTime;
		}

		refreshParticles();
		animationFrame = requestAnimationFrame(loop);
	}

	onMount(() => {
		if (!root) return;

		container = getContainer();
		incrementInstances();

		const isTouchInteraction = "ontouchstart" in window;
		const tap = isTouchInteraction ? "touchstart" : "mousedown";
		const tapEnd = isTouchInteraction ? "touchend" : "mouseup";
		const move = isTouchInteraction ? "touchmove" : "mousemove";

		root.addEventListener(move, updateMousePosition, { passive: true });
		root.addEventListener(tap, tapHandler, { passive: true });
		root.addEventListener(tapEnd, disableAutoAddParticle, { passive: true });
		root.addEventListener("mouseleave", disableAutoAddParticle, {
			passive: true,
		});

		loop();

		return () => {
			root?.removeEventListener(move, updateMousePosition);
			root?.removeEventListener(tap, tapHandler);
			root?.removeEventListener(tapEnd, disableAutoAddParticle);
			root?.removeEventListener("mouseleave", disableAutoAddParticle);
		};
	});

	onDestroy(() => {
		if (typeof window === "undefined") return;

		const interval = window.setInterval(() => {
			if (animationFrame && particles.length === 0) {
				cancelAnimationFrame(animationFrame);
				window.clearInterval(interval);
				decrementInstances();
			}
		}, 500);
	});
</script>

<span bind:this={root} class="cool-mode">
	<slot />
</span>

<style>
	.cool-mode {
		display: inline-block;
		position: relative;
	}
</style>
