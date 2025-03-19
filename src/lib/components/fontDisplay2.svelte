<script lang="ts">
	import type opentype from 'opentype.js';
	import InputNumber from '$lib/ui/inputNumber.svelte';
	import Label from '$lib/ui/label.svelte';
	import { nanoid } from 'nanoid';
	import Button from '$lib/ui/button.svelte';
	import { onMount, afterUpdate } from 'svelte';

	export let font: opentype.Font;
	export let text: string;

	let canvasWidth = 900;
	let canvasHeight = 300;
	let fontSize = canvasHeight / 4;
	let x = fontSize / 5;
	let y = fontSize / 5;

	let canvas: HTMLCanvasElement;
	let container: HTMLDivElement;
	let resizeObserver: ResizeObserver;
	let shouldRender = true;

	onMount(() => {
		resizeObserver = new ResizeObserver((entries) => {
			const { width, height } = entries[0].contentRect;
			let needsRender = false;

			if (width > 0 && width !== canvasWidth) {
				canvasWidth = width;
				needsRender = true;
			}

			// Also track height changes if we want full height
			if (height > 0 && height !== canvasHeight) {
				// Leave some space for controls
				canvasHeight = height - 80; // Accounting for control panel height
				needsRender = true;
			}

			if (needsRender) {
				shouldRender = true;
			}
		});

		if (container) {
			resizeObserver.observe(container);
		}

		return () => {
			if (resizeObserver) resizeObserver.disconnect();
		};
	});

	// Reactively track changes and mark for rendering
	$: if (font) shouldRender = true;
	$: if (text) shouldRender = true;
	$: if (fontSize) shouldRender = true;
	$: if (x) shouldRender = true;
	$: if (y) shouldRender = true;
	$: if (canvasWidth) shouldRender = true;
	$: if (canvasHeight) shouldRender = true;

	// Perform rendering when needed after update
	afterUpdate(() => {
		if (shouldRender && canvas && font) {
			render(font, canvas, text, fontSize, x, y, canvasWidth, canvasHeight);
			shouldRender = false;
		}
	});

	function render(
		font: opentype.Font,
		canvas: HTMLCanvasElement,
		text: string,
		fontSize: number,
		x: number,
		y: number,
		canvasWidth: number,
		canvasHeight: number
	) {
		try {
			if (!canvas) return;
			if (!font || !font.draw) {
				console.error('Invalid font object or missing draw method');
				return;
			}

			const ctx = canvas.getContext('2d', {
				willReadFrequently: true
			}) as CanvasRenderingContext2D;

			if (!ctx) return;
			ctx.reset();

			// Clear the canvas
			ctx.clearRect(0, 0, canvasWidth, canvasHeight);

			// Handle empty text
			if (!text) {
				ctx.font = '16px sans-serif';
				ctx.fillText('No text to render', 20, 40);
				return;
			}

			try {
				// Calculate font metrics for proper positioning
				const fontAscent = (font.ascender / font.unitsPerEm) * fontSize;
				const topMargin = y; // Use y as the top margin

				// Start from top + margin + ascent (to position baseline correctly)
				let lineY = topMargin + fontAscent;
				const lineHeight = fontSize * 1.2; // Add some space between lines

				// Implement text wrapping
				const words = text.split(' ');
				let line = '';

				for (let i = 0; i < words.length; i++) {
					const testLine = line + (line ? ' ' : '') + words[i];

					// Try to get width with kerning, fall back to simpler calculation if it fails
					let testWidth;
					try {
						testWidth = font.getAdvanceWidth(testLine, fontSize, { kerning: false });
					} catch (e) {
						// Fallback: calculate width by summing individual glyph widths
						testWidth = 0;
						for (let j = 0; j < testLine.length; j++) {
							const glyphIndex = font.charToGlyphIndex(testLine[j]);
							const glyph = font.glyphs.get(glyphIndex);
							if (glyph && glyph.advanceWidth !== undefined) {
								testWidth += (glyph.advanceWidth * fontSize) / font.unitsPerEm;
							}
						}
					}

					if (testWidth > canvasWidth - x * 2 && i > 0) {
						// If this line would be too wide, draw the previous line and start a new one
						font.draw(ctx, line, x, lineY, fontSize, { kerning: false });
						line = words[i];
						lineY += lineHeight; // Move down for next line
					} else {
						line = testLine;
					}
				}

				// Draw the last line
				font.draw(ctx, line, x, lineY, fontSize, { kerning: false });
			} catch (renderError) {
				console.error('Error rendering text:', renderError);
				// Fallback to canvas text rendering
				ctx.font = `${fontSize}px sans-serif`;
				ctx.fillText(text, x, y + fontSize);
			}
		} catch (e) {
			console.error('Error rendering font:', e);
		}
	}

	const id = nanoid(5);
</script>

<div bind:this={container} class="flex h-full w-full min-w-[600px] flex-col">
	<div class="mb-4 flex flex-wrap items-end gap-4">
		<div class="flex max-w-[300px] flex-grow items-end">
			<Label target={`${id}-size`} class_="mr-2">Font size: {fontSize}px</Label>
			<input
				type="range"
				id={`${id}-size`}
				name={`${id}-size`}
				bind:value={fontSize}
				min="10"
				max="200"
				step="1"
				class="h-8 w-full cursor-pointer bg-slate-200 accent-slate-800"
			/>
		</div>
		<div>
			<Label target={`${id}-x`}>Text X</Label>
			<InputNumber name={`${id}-x`} bind:value={x} />
		</div>
		<div>
			<Label target={`${id}-y`}>Text Y</Label>
			<InputNumber name={`${id}-y`} bind:value={y} />
		</div>
	</div>

	<div class="flex-grow">
		<canvas
			bind:this={canvas}
			class="h-full w-full bg-gray-100"
			width={canvasWidth}
			height={canvasHeight}
		/>
	</div>
</div>
