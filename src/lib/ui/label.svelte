<script lang="ts">
	export let target: string;
	export let class_ = ''; // Use class_ to avoid conflict with reserved keyword
</script>

<!--  -->

<label class="block font-mono text-sm text-slate-700 {class_}" for={target}>
	<slot />
</label>
