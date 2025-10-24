<script lang="ts">
	import { t } from "svelte-i18n";
	import { is_dangerous } from "../sql";
	import { export_csv } from "$lib/csv";
	import Icon from "@iconify/svelte";

	export let database: string;
	export let table: string;

	$: query = `SELECT * FROM \`${table}\` LIMIT 100`;
	$: danger = is_dangerous(query);
	$: isEditable = isEditableQuery(query);

	let running = false;
	let result: D1Result<any> | undefined;
	let error: string | undefined;
	let locked = true;

	function isEditableQuery(query: string): boolean {
		// Check if it's a simple SELECT from a single table (no JOINs, subqueries, etc.)
		const trimmed = query.trim().toLowerCase();
		return (
			trimmed.startsWith("select") &&
			!trimmed.includes("join") &&
			!trimmed.includes("union") &&
			!trimmed.includes("(") &&
			!trimmed.includes("group by") &&
			!trimmed.includes("having")
		);
	}

	function getQueryWithRowid(query: string): string {
		if (!isEditableQuery(query)) return query;

		// If it's a simple SELECT, add rowid to make it editable
		if (query.toLowerCase().includes("select *")) {
			return query.replace(/select \*/i, "SELECT rowid AS _, *");
		}
		return query;
	}

	async function run() {
		if (running) {
			return;
		}
		running = true;

		try {
			const queryWithRowid = getQueryWithRowid(query);
			const res = await fetch(`/api/db/${database}/all`, {
				method: "POST",
				body: JSON.stringify({ query: queryWithRowid }),
			});

			const json = await res.json<D1Result | { message: string }>();
			if (json) {
				if ("message" in json) {
					throw new Error(json.message);
				}
				result = json;
				error = undefined;
			} else {
				throw new Error($t("plugin.run-query.no-result"));
			}
		} catch (err) {
			result = undefined;
			if (err instanceof Error) {
				error = err.message;
			} else {
				error = $t("plugin.run-query.unknown-error");
			}
		} finally {
			running = false;
			setTimeout(() => {
				document.querySelector("#bottom")?.scrollIntoView({ behavior: "smooth" });
			}, 50);
		}
	}

	function handler(evt: KeyboardEvent) {
		if (evt.code === "Enter" && evt.shiftKey === true) {
			run();
		}
	}

	async function edit(rowid: unknown, col: string) {
		if (running || locked) {
			return;
		}
		running = true;

		const record = result?.results?.find((r) => r._ === rowid);

		console.log("edit", rowid, col, record);

		try {
			if (typeof rowid !== "number") {
				throw new Error($t("plugin.table-browser.invalid-rowid"));
			}
			if (!record) {
				throw new Error($t("plugin.table-browser.no-record"));
			}
			const res = await fetch(`/api/db/${database}/${table}/data/?rowid=${rowid}`, {
				method: "PUT",
				headers: {
					"Content-Type": "application/json",
				},
				body: JSON.stringify({
					...record,
					_: undefined,
				}),
			});
			const json = await res.json<any>();
			if (json) {
				if ("error" in json) {
					error = json.error?.message || "Unknown error";
				} else {
					error = undefined;
				}
			} else {
				throw new Error($t("plugin.table-browser.no-result"));
			}
		} catch (err) {
			error = err instanceof Error ? err.message : $t("plugin.table-browser.unknown-error");
		} finally {
			running = false;
		}
	}
</script>

<div class="w-full">
	<textarea
		class="textarea-border textarea h-24 w-full resize-y font-mono"
		placeholder="SELECT COUNT(*) AS c FROM {table}"
		bind:value={query}
		on:keypress={handler}
	></textarea>
</div>

<button class="btn-primary btn" class:btn-error={danger} on:click={run} disabled={running}
	>{$t("plugin.run-query.run")}</button
>

{#if isEditable}
	<div class="pt-4 pb-2">
		<label class="swap">
			<input type="checkbox" bind:checked={locked} />
			<div class="swap-on flex items-center gap-2">
				<Icon icon="mdi:lock-outline" class="inline-block text-xl" />
				{$t("plugin.table-browser.table-is-locked-click-to-unlock")}
			</div>
			<div class="swap-off flex items-center gap-2">
				<Icon icon="mdi:lock-open-outline" class="inline-block text-xl" />
				{$t("plugin.table-browser.table-is-unlocked-click-to-lock")}
			</div>
		</label>
	</div>
{:else}
	<div class="pt-4 pb-2">
		<div class="alert alert-info">
			<Icon icon="mdi:information" class="inline-block text-xl" />
			<span
				>Complex queries cannot be edited. Use simple SELECT statements to enable editing.</span
			>
		</div>
	</div>
{/if}

{#if result}
	<div class="divider"></div>

	{#if result?.results?.length}
		<div class="max-h-[80vh] overflow-auto transition-opacity" class:opacity-50={running}>
			<table class="table-sm table min-w-full">
				<thead>
					<tr class="bg-base-200 sticky top-0 z-10 shadow">
						{#each Object.keys(result.results[0]) as key}
							<th class="!relative normal-case">{key}</th>
						{/each}
					</tr>
				</thead>
				<tbody>
					{#each result.results as row}
						<tr class="group hover">
							{#each Object.keys(row) as key}
								{#if key !== "_"}
									<td>
										{#if isEditable}
											{#if typeof row[key] === "number"}
												<input
													class="input-ghost input input-xs hover:input-border text-base transition-all disabled:bg-transparent"
													type="number"
													bind:value={row[key]}
													on:blur={() => edit(row._, key)}
													disabled={locked || running}
													title={locked
														? $t("plugin.table-browser.table-is-locked")
														: undefined}
												/>
											{:else}
												<input
													class="input-ghost input input-xs hover:input-border text-base transition-all disabled:bg-transparent"
													bind:value={row[key]}
													on:change={() => edit(row._, key)}
													disabled={locked || running}
													title={locked
														? $t("plugin.table-browser.table-is-locked")
														: undefined}
												/>
											{/if}
										{:else}
											<span
												class="text-base-content"
												class:text-opacity-50={row[key] === null}
												>{row[key]}</span
											>
										{/if}
									</td>
								{/if}
							{/each}
						</tr>
					{/each}
				</tbody>
			</table>
		</div>
	{:else}
		<p>
			{$t("plugin.run-query.no-results")}
		</p>
	{/if}

	<div class="mt-2 flex w-full justify-between gap-2 space-x-2">
		<p class="text-sm opacity-70">
			{$t("plugin.run-query.n-ms-m-changes", {
				values: {
					n: result.meta.duration.toFixed(2),
					rr: result.meta.rows_read ?? "x",
					rw: result.meta.rows_written ?? result.meta.changes,
				},
			})}
		</p>
		{#if result?.results?.length}
			<button
				class="btn-primary btn-outline btn-sm btn"
				on:click={() => (result ? export_csv(result.results, table) : undefined)}
			>
				{$t("plugin.run-query.export")}
			</button>
		{/if}
	</div>
{/if}

{#if error}
	<div class="divider"></div>

	<div class="alert alert-error shadow-lg">
		<div>{error}</div>
	</div>
{/if}

<div id="bottom"></div>
