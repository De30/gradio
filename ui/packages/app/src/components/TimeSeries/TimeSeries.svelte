<script lang="ts">
	import { Upload } from "@gradio/upload";
	import type { FileData } from "@gradio/upload";
	import { Chart } from "@gradio/chart";
	import { _ } from "svelte-i18n";

	function format_value(val: StaticData) {
		return val.data.map((r) =>
			r.reduce((acc, next, i) => ({ ...acc, [val.headers[i]]: next }), {})
		);
	}

	interface StaticData {
		data: Array<Array<number>>;
		headers: Array<string>;
	}
	interface Data {
		data: Array<Array<number>> | string;
		headers?: Array<string>;
	}

	export let value: null | Data;
	export let theme: string;
	export let y: Array<string>;
	export let x: string;
	export let is_static: boolean;

	let _value: string | null;

	function data_uri_to_blob(data_uri: string) {
		const byte_str = atob(data_uri.split(",")[1]);
		const mime_str = data_uri.split(",")[0].split(":")[1].split(";")[0];

		const ab = new ArrayBuffer(byte_str.length);
		const ia = new Uint8Array(ab);

		for (let i = 0; i < byte_str.length; i++) {
			ia[i] = byte_str.charCodeAt(i);
		}

		return new Blob([ab], { type: mime_str });
	}

	function blob_to_string(blob: Blob) {
		const reader = new FileReader();

		reader.addEventListener("loadend", (e) => {
			//@ts-ignore
			_value = e.srcElement.result;
		});

		reader.readAsText(blob);
	}

	$: {
		if (value && value.data && typeof value.data === "string") {
			if (!value) _value = null;
			else blob_to_string(data_uri_to_blob(value.data));
		}
	}

	interface XRow {
		name: string;
		values: Array<number>;
	}

	interface YRow {
		name: string;
		values: Array<{ x: number; y: number }>;
	}

	function make_dict(x: XRow, y: Array<YRow>): Data {
		const headers = [];
		const data = [];

		headers.push(x.name);
		y.forEach(({ name }) => headers.push(name));

		for (let i = 0; i < x.values.length; i++) {
			let _data = [];
			_data.push(x.values[i]);
			y.forEach(({ values }) => _data.push(values[i].y));

			data.push(_data);
		}
		return { headers, data };
	}

	function handle_load(v: string | FileData | (string | FileData)[] | null) {
		value = { data: v as string };
		// setValue({ data: v as string });
		return v;
	}

	$: _value = value == null ? null : _value;
	$: static_data = is_static && format_value(value as StaticData);
</script>

{#if is_static && static_data}
	<Chart value={static_data} />
{:else}
	{#if _value}
		<Chart
			value={_value}
			{y}
			{x}
			on:process={({ detail: { x, y } }) => (value = make_dict(x, y))}
		/>
	{/if}
	{#if value == null}
		<Upload
			filetype="text/csv"
			on:load={({ detail }) => handle_load(detail)}
			include_file_metadata={false}
			{theme}
		>
			{$_("interface.drop_csv")}
			<br />- {$_("interface.or")} -<br />
			{$_("interface.click_to_upload")}
		</Upload>
	{/if}
{/if}