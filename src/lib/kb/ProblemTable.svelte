<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { PROBLEMS, problemOf, uniqueById, type Problem, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // `tickets`: the ticket filed for each problem, by number, in `repo`.
  let { ticket, tickets, repo }: { ticket: number; tickets: Record<Problem, number>; repo: string } = $props();

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const rows = $derived(
    [...PROBLEMS, null].map((problem) => {
      const rs = merged.filter((r) => problemOf(r, tenantOf(r), notes.oneDayMove) === problem);
      const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
      return {
        problem,
        reports: rs.length,
        tenants: new Set(rs.map((r) => r.tenant)).size,
        channels: [...new Set(rs.map((r) => r.channel))].sort().join(' · '),
        seen: rs.length ? `${days[0]} → ${days[days.length - 1]}` : '',
      };
    }),
  );
  const filed = $derived(rows.filter((r) => r.problem && r.reports));
  const unplaced = $derived(rows.find((r) => r.problem === null)?.reports ?? 0);
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} (<strong>{(reports as Report[]).length}</strong> rows in the export) ·
  <strong>{filed.length}</strong> problems · one ticket each
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>channels</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each filed as r (r.problem)}
      <tr>
        <td><code>{r.problem}</code></td>
        <td>{r.reports}</td>
        <td>{r.tenants}</td>
        <td>{r.channels}</td>
        <td>{r.seen}</td>
        <td><a href="{repo}/issues/{tickets[r.problem as Problem]}">#{tickets[r.problem as Problem]}</a></td>
      </tr>
    {/each}
  </tbody>
</table>

{#if unplaced}
  <p class="warn">{unplaced} reports fit no problem: their tenant is not on file or writes the month first. They have no ticket yet.</p>
{/if}

<style>
  .sum { font-size: 1.1rem; }
  .warn { color: var(--warn); font-weight: 600; }
</style>
