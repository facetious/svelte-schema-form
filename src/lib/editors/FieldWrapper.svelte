<script lang="ts">
	import { jsonPointerToPath, schemaLabel } from "../types/schema";
	import type { CommonComponentParameters } from "../types/CommonComponentParameters";
	import { stringToHtml } from "../utilities.js";

	export let params: CommonComponentParameters;
	export let schema: any;

	const title = schemaLabel(schema, params.path);
	const id = jsonPointerToPath(params.path.join('/'));
	$: error = params.validationErrors[id];
</script>

{#if params.containerParent !== "array"}
	<label id={`label-${id}`} for={id} class:required={params.required} class:readonly={schema.readOnly || params.containerReadOnly}>
		{@html stringToHtml(title)}
		{#if schema.description}
			<span class="info" title={schema.description}></span>
		{/if}
	</label>
{/if}
<slot></slot>
{#if error && params.showErrors}
	<div class="error">{error}</div>
{/if}