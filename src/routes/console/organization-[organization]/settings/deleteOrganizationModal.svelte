<script lang="ts">
    import { Modal } from '$lib/components';
    import { Button } from '$lib/elements/forms';
    import { addNotification } from '$lib/stores/notifications';
    import { sdk } from '$lib/stores/sdk';
    import { organization, organizationList } from '$lib/stores/organization';
    import { goto, invalidate } from '$app/navigation';
    import { base } from '$app/paths';
    import { Dependencies } from '$lib/constants';
    import { Submit, trackEvent, trackError } from '$lib/actions/analytics';
    import { projects } from '../store';
    import { toLocaleDate } from '$lib/helpers/date';
    import { isCloud } from '$lib/system';
    import type { InvoiceList } from '$lib/sdk/billing';

    export let showDelete = false;
    export let invoices: InvoiceList;
    let error: string = null;

    async function deleteOrg() {
        try {
            if (isCloud) {
                await sdk.forConsole.billing.deleteOrganization($organization.$id);
            } else {
                await sdk.forConsole.teams.delete($organization.$id);
            }

            await invalidate(Dependencies.ACCOUNT);
            await invalidate(Dependencies.ORGANIZATION);
            trackEvent(Submit.OrganizationDelete);
            if ($organizationList?.total > 1) {
                const account = await sdk.forConsole.account.get();
                const teamId =
                    account.prefs.organization ??
                    $organizationList.teams[0].$id === $organization.$id
                        ? $organizationList.teams[1].$id
                        : $organizationList.teams[0].$id;

                await goto(`${base}/console/organization-${teamId}`);
            } else {
                await goto(`${base}/console/onboarding`);
            }
            showDelete = false;
            addNotification({
                type: 'success',
                message: `${$organization.name} has been flagged for deletion`
            });
        } catch (e) {
            error = e.message;
            trackError(e, Submit.OrganizationDelete);
        }
    }

    $: upcomingInvoice = invoices?.invoices.find((i) => i.status === 'upcoming' && i.amount > 0);
</script>

<Modal
    title="Delete organization"
    onSubmit={deleteOrg}
    bind:show={showDelete}
    bind:error
    icon="exclamation"
    state="warning"
    headerDivider={false}>
    {#if upcomingInvoice}
        <p data-private>
            The organization <b>{$organization.name}</b> will be flagged for deletion.
        </p>

        <p data-private>
            All existing projects will be paused and the organization will be deleted once your
            upcoming invoice is processed on {toLocaleDate($organization.billingNextInvoiceDate)}.
        </p>
    {:else}
        <p data-private>
            Are you sure you want to delete <b>{$organization.name}</b>? All projects ({$projects.total})
            and data associated with this organization will be deleted. This action is irreversible.
        </p>
    {/if}
    <svelte:fragment slot="footer">
        <Button text on:click={() => (showDelete = false)}>Cancel</Button>
        <Button secondary submit>Delete</Button>
    </svelte:fragment>
</Modal>
