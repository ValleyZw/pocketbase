<script>
    import { Collection } from "pocketbase";
    import { createEventDispatcher, tick } from "svelte";
    import { scale } from "svelte/transition";
    import CommonHelper from "@/utils/CommonHelper";
    import ApiClient from "@/utils/ApiClient";
    import { errors, setErrors } from "@/stores/errors";
    import { confirm } from "@/stores/confirmation";
    import { addSuccessToast } from "@/stores/toasts";
    import { addCollection, removeCollection, activeCollection } from "@/stores/collections";
    import tooltip from "@/actions/tooltip";
    import Field from "@/components/base/Field.svelte";
    import Toggler from "@/components/base/Toggler.svelte";
    import OverlayPanel from "@/components/base/OverlayPanel.svelte";
    import CollectionFieldsTab from "@/components/collections/CollectionFieldsTab.svelte";
    import CollectionRulesTab from "@/components/collections/CollectionRulesTab.svelte";
    import CollectionUpdateConfirm from "@/components/collections/CollectionUpdateConfirm.svelte";

    const TAB_FIELDS = "fields";
    const TAB_RULES = "api_rules";
    const dispatch = createEventDispatcher();

    let collectionPanel;
    let confirmChangesPanel;

    let original = null;
    let collection = new Collection();
    let isSaving = false;
    let confirmClose = false; // prevent close recursion
    let activeTab = TAB_FIELDS;
    let initialFormHash = calculateFormHash(collection);

    $: schemaTabError =
        // extract the direct schema field error, otherwise - return a generic message
        typeof CommonHelper.getNestedVal($errors, "schema.message", null) === "string"
            ? CommonHelper.getNestedVal($errors, "schema.message")
            : "Has errors";

    $: isSystemUpdate = !collection.isNew && collection.system;

    $: hasChanges = initialFormHash != calculateFormHash(collection);

    $: canSave = collection.isNew || hasChanges;

    export function changeTab(newTab) {
        activeTab = newTab;
    }

    export function show(model) {
        load(model);

        confirmClose = true;

        changeTab(TAB_FIELDS);

        return collectionPanel?.show();
    }

    export function hide() {
        return collectionPanel?.hide();
    }

    async function load(model) {
        setErrors({}); // reset errors
        if (typeof model !== "undefined") {
            original = model;
            collection = model?.clone();
        } else {
            original = null;
            collection = new Collection();
        }
        // normalize
        collection.schema = collection.schema || [];
        collection.originalName = collection.name || "";

        await tick();

        initialFormHash = calculateFormHash(collection);
    }

    function saveWithConfirm() {
        if (collection.isNew) {
            return save();
        } else {
            confirmChangesPanel?.show(collection);
        }
    }

    function save() {
        if (isSaving) {
            return;
        }

        isSaving = true;

        const data = exportFormData();

        let request;
        if (collection.isNew) {
            request = ApiClient.Collections.create(data);
        } else {
            request = ApiClient.Collections.update(collection.id, data);
        }

        request
            .then((result) => {
                confirmClose = false;
                hide();
                addSuccessToast(
                    collection.isNew ? "Successfully created collection." : "Successfully updated collection."
                );
                addCollection(result);

                if (collection.isNew) {
                    $activeCollection = result;
                }

                dispatch("save", result);
            })
            .catch((err) => {
                ApiClient.errorResponseHandler(err);
            })
            .finally(() => {
                isSaving = false;
            });
    }

    function exportFormData() {
        const data = collection.export();
        data.schema = data.schema.slice(0);

        // remove deleted fields
        for (let i = data.schema.length - 1; i >= 0; i--) {
            const field = data.schema[i];
            if (field.toDelete) {
                data.schema.splice(i, 1);
            }
        }

        return data;
    }

    function deleteConfirm() {
        if (!original?.id) {
            return; // nothing to delete
        }

        confirm(`Do you really want to delete collection "${original?.name}" and all its records?`, () => {
            return ApiClient.Collections.delete(original?.id)
                .then(() => {
                    hide();
                    addSuccessToast(`Successfully deleted collection "${original?.name}".`);
                    dispatch("delete", original);
                    removeCollection(original);
                })
                .catch((err) => {
                    ApiClient.errorResponseHandler(err);
                });
        });
    }

    function calculateFormHash(m) {
        return JSON.stringify(m);
    }
</script>

<OverlayPanel
    bind:this={collectionPanel}
    class="overlay-panel-lg colored-header collection-panel"
    beforeHide={() => {
        if (hasChanges && confirmClose) {
            confirm("You have unsaved changes. Do you really want to close the panel?", () => {
                confirmClose = false;
                hide();
            });
            return false;
        }
        return true;
    }}
    on:hide
    on:show
>
    <svelte:fragment slot="header">
        <h4>
            {collection.isNew ? "New collection" : "Edit collection"}
        </h4>

        {#if !collection.isNew && !collection.system}
            <div class="flex-fill" />
            <button type="button" class="btn btn-sm btn-circle btn-secondary flex-gap-0">
                <i class="ri-more-line" />
                <Toggler class="dropdown dropdown-right m-t-5">
                    <button type="button" class="dropdown-item closable" on:click={() => deleteConfirm()}>
                        <i class="ri-delete-bin-7-line" />
                        <span class="txt">Delete</span>
                    </button>
                </Toggler>
            </button>
        {/if}

        <form
            class="block"
            on:submit|preventDefault={() => {
                canSave && saveWithConfirm();
            }}
        >
            <Field
                class="form-field required m-b-0 {isSystemUpdate ? 'disabled' : ''}"
                name="name"
                let:uniqueId
            >
                <label for={uniqueId}>Name</label>
                <!-- svelte-ignore a11y-autofocus -->
                <input
                    type="text"
                    id={uniqueId}
                    required
                    disabled={isSystemUpdate}
                    spellcheck="false"
                    autofocus={collection.isNew}
                    placeholder={`eg. "posts"`}
                    value={collection.name}
                    on:input={(e) => {
                        collection.name = CommonHelper.slugify(e.target.value);
                        e.target.value = collection.name;
                    }}
                />
                {#if collection.system}
                    <div class="help-block">System collection</div>
                {/if}
            </Field>

            <input type="submit" class="hidden" tabindex="-1" />
        </form>

        <div class="tabs-header stretched">
            <button
                type="button"
                class="tab-item"
                class:active={activeTab === TAB_FIELDS}
                on:click={() => changeTab(TAB_FIELDS)}
            >
                <span class="txt">Fields</span>
                {#if !CommonHelper.isEmpty($errors?.schema)}
                    <i
                        class="ri-error-warning-fill txt-danger"
                        transition:scale|local={{ duration: 150, start: 0.7 }}
                        use:tooltip={schemaTabError}
                    />
                {/if}
            </button>

            <button
                type="button"
                class="tab-item"
                class:active={activeTab === TAB_RULES}
                on:click={() => changeTab(TAB_RULES)}
            >
                <span class="txt">API Rules</span>
                {#if !CommonHelper.isEmpty($errors?.listRule) || !CommonHelper.isEmpty($errors?.viewRule) || !CommonHelper.isEmpty($errors?.createRule) || !CommonHelper.isEmpty($errors?.updateRule) || !CommonHelper.isEmpty($errors?.deleteRule)}
                    <i
                        class="ri-error-warning-fill txt-danger"
                        transition:scale|local={{ duration: 150, start: 0.7 }}
                        use:tooltip={"Has errors"}
                    />
                {/if}
            </button>
        </div>
    </svelte:fragment>

    <div class="tabs-content">
        <!-- avoid rerendering the fields tab -->
        <div class="tab-item" class:active={activeTab === TAB_FIELDS}>
            <CollectionFieldsTab bind:collection />
        </div>

        {#if activeTab === TAB_RULES}
            <div class="tab-item active">
                <CollectionRulesTab bind:collection />
            </div>
        {/if}
    </div>

    <svelte:fragment slot="footer">
        <button type="button" class="btn btn-secondary" disabled={isSaving} on:click={() => hide()}>
            <span class="txt">Cancel</span>
        </button>
        <button
            type="button"
            class="btn btn-expanded"
            class:btn-loading={isSaving}
            disabled={!canSave || isSaving}
            on:click={() => saveWithConfirm()}
        >
            <span class="txt">{collection.isNew ? "Create" : "Save changes"}</span>
        </button>
    </svelte:fragment>
</OverlayPanel>

<CollectionUpdateConfirm bind:this={confirmChangesPanel} on:confirm={() => save()} />

<style>
    .tabs-content {
        z-index: 3; /* autocomplete dropdown overlay fix */
    }
</style>
