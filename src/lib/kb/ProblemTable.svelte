<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type CauseGroup, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // The desk's own count (CauseTable): every report under the cause it is filed by, and the reports the
  // desk read as a one-day move (desk-notes.json) counted again as their own problem, footnoted where filed.
  type Problem = CauseGroup['cause'] | 'timing';
  let { ticket, tickets, repo }: { ticket: number; tickets: Partial<Record<Problem, number>>; repo: string } = $props();

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const groups = $derived(groupByCause(merged, tenantOf));
  const moved = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));

  const row = (problem: Problem, rs: Report[]) => {
    const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
    return {
      problem,
      reports: rs.length,
      tenants: new Set(rs.map((r) => r.tenant)).size,
      channels: [...new Set(rs.map((r) => r.channel))].sort().join(' · '),
      seen: `${days[0]} → ${days[days.length - 1]}`,
      issue: tickets[problem],
    };
  };
  const rows = $derived([...groups.map((g) => row(g.cause, g.reports)), ...(moved.length ? [row('timing', moved)] : [])]);
  const filedUnder = $derived(groups.filter((g) => readAs(g, notes.oneDayMove)).map((g) => g.cause));
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} (<strong>{(reports as Report[]).length}</strong> rows in the export) ·
  <strong>{rows.length}</strong> problems · one ticket each
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>channels</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as r (r.problem)}
      <tr>
        <td><code>{r.problem}</code></td>
        <td>{r.reports}</td>
        <td>{r.tenants}</td>
        <td>{r.channels}</td>
        <td>{r.seen}</td>
        <td>{#if r.issue}<a href="{repo}/issues/{r.issue}">#{r.issue}</a>{:else}<span class="warn">no ticket</span>{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

{#if moved.length}
  <p class="note">
    The {moved.length} <code>timing</code> reports are also counted under <code>{filedUnder.join(', ')}</code>, where they
    are filed by tenant, as the desk counts them.
  </p>
{/if}

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
  .warn { color: var(--warn); font-weight: 600; }
</style>
