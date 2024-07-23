<svelte:options accessors />

<script lang="ts">
	import { createEventDispatcher, onMount } from "svelte";
	import SubSchemaForm from "./SubSchemaForm.svelte";
	import String from "./editors/String.svelte";
	import FieldWrapper from './editors/FieldWrapper.svelte';
	import ObjectEditor from "./editors/Object.svelte";
	import set from "lodash-es/set";
	import get from "lodash-es/get";
	import { validator } from "@exodus/schemasafe";
	import { FileNone, type CommonComponentParameters, type ValidationErrors } from './types/CommonComponentParameters';
	import Enum from './editors/Enum.svelte';
	import Array from './editors/Array.svelte';
	import { incr, nullOptionalsAllowed } from './utilities.js';
	import Boolean from './editors/Boolean.svelte';
	import Number from './editors/Number.svelte';
	import { errorMapper } from "./errorMapper";
	import Upload from "./editors/Upload.svelte";
	import TextArea from "./editors/TextArea.svelte";
	import ArrayBlocks from "./editors/ArrayBlocks.svelte";
    import Autocomplete from "./editors/Autocomplete.svelte";
	import Hidden from "./editors/Hidden.svelte";
    import ListDetail from "./editors/ListDetail.svelte";
    import Currency from "./editors/Currency.svelte";
    import Radio from "./editors/Radio.svelte";

	export let schema: any;
	export let value: any;
	export let uploadFiles: Record<string, FileList> = {};
	export let dirty: boolean = false;
	export let showErrors: boolean = true;
	export let collapsible: boolean = false;
	export let components: Record<string, new (...args: any[]) => any> = {};
	export let validationErrors = {} as ValidationErrors;
	export let componentContext: Record<string, unknown> = {};
	export let editorForSchema: (schema: any, originalEditorForSchema: (schema: any) => string) => string | undefined = (schema, original) => original(schema);

	const dispatch = createEventDispatcher();

	$: valueForValidation = value;

	$: composedValidationErrors = revalidate(validationErrors, valueForValidation);

	const revalidate = (externalValidationErrors: ValidationErrors, newValue?: any) => {
		const validate = validator(nullOptionalsAllowed(schema), { includeErrors: true, allErrors: true, allowUnusedKeywords: true, formats: { "date-time": (val) => !isNaN(new Date(val).getTime()) } });
		const validatorResult = validate(newValue || value);
		return {
			...(externalValidationErrors || {}),
			...Object.fromEntries(
			(validate.errors || []).map(ve => errorMapper(schema, value, ve.keywordLocation, ve.instanceLocation)))
		};
	}

	onMount(() => {
		if (Object.keys(composedValidationErrors).length > 0) {
			// set initial errors
			dispatch('value', {
				path: [], value, errors: composedValidationErrors
			});
		}
	});

	let params: CommonComponentParameters;
	$: params = {
		value,
		files: uploadFiles,
		path: [],
		components: Object.assign({
			string: String,
			password: String,
			email: String,
			time: String,
			"date-time": String,
			date: String,
			"string-ipv4": String,
			number: Number,
			integer: Number,
			boolean: Boolean,
			fieldWrapper: FieldWrapper,
			object: ObjectEditor,
			array: Array,
			enum: Enum,
			upload: Upload,
			textarea: TextArea,
			hidden: Hidden,
			blocks: ArrayBlocks,
			autocomplete: Autocomplete,
			"list-detail": ListDetail,
			currency: Currency,
			radio: Radio
		}, components),
		componentContext,
		pathChanged,
		validationErrors: composedValidationErrors,
		containerParent: "none",
		containerReadOnly: schema.readOnly || false,
		showErrors,
		collapsible,
		idx: incr(),
		editorForSchema,
	} as CommonComponentParameters;

	const pathChanged = (path: string[], val: any, op?: string, type?: string) => {
		let changed = false;

		if (val instanceof FileList) {
			uploadFiles[path.join('.')] = val;
			dirty = true;
			return val;
		} else if (val === FileNone) {
			delete uploadFiles[path.join('.')];
			dirty = true;
			return val;
		}

		const curr = path.length === 0 ? params.value : get(params.value, path);
		if (val === curr && op !== "innerSubmit") return;

		if (val === undefined && path.length > 0) {
			const pathFront = path.slice(0, -1);
			const parent = pathFront.length ? get(params.value, path.slice(0, -1)) : params.value;
			delete parent[path[path.length - 1]];
		} else {
			if (path.length === 0) {
				params.value = val;
			} else {
				if (type === 'datetime-local') {
					val = new Date(new Date(val).getTime() - new Date().getTimezoneOffset() * 60_000).toISOString();
				}
				set(params.value, path, val);
			}
		}

		valueForValidation = params.value;

		const succeeded = dispatch('value', {
			path, pathValue: val, value: params.value, errors: composedValidationErrors, op
		}, { cancelable: true });

		console.log(`dispatch value path: ${path.join('.')} val: ${JSON.stringify(val)},${op ? " op: " + op : ''} errors: ${JSON.stringify(composedValidationErrors)}, succeeded: ${succeeded}`);

		// update if value event not cancelled.
		if (succeeded) {
			value = params.value;
			dirty = true;
		} else {
			valueForValidation = value;
		}
		return val;
	};
</script>

<SubSchemaForm {params} {value} bind:schema />
