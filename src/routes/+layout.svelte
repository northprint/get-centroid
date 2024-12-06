<script lang="ts">
	import '../app.css';
	let { children } = $props();
</script>
{@render children()}
<!-- ベースの背景は薄いグレーにする -->
<style>
	:global(body) {
		background-color: #f8f9fa;
	}
</style>