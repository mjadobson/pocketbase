<script>
    import { createEventDispatcher } from "svelte";
    import { slide } from "svelte/transition";
    import { Admin } from "pocketbase";
    import CommonHelper from "@/utils/CommonHelper";
    import ApiClient from "@/utils/ApiClient";
    import tooltip from "@/actions/tooltip";
    import { setErrors } from "@/stores/errors";
    import { confirm } from "@/stores/confirmation";
    import { addSuccessToast } from "@/stores/toasts";
    import Field from "@/components/base/Field.svelte";
    import Toggler from "@/components/base/Toggler.svelte";
    import OverlayPanel from "@/components/base/OverlayPanel.svelte";

    const dispatch = createEventDispatcher();
    const formId = "admin_" + CommonHelper.randomString(5);

    let panel;
    let admin = new Admin();
    let isSaving = false;
    let confirmClose = false; // prevent close recursion
    let avatar = 0;
    let email = "";
    let password = "";
    let passwordConfirm = "";
    let changePasswordToggle = false;

    $: hasChanges =
        (admin.isNew && email != "") ||
        changePasswordToggle ||
        email !== admin.email ||
        avatar !== admin.avatar;

    export function show(model) {
        load(model);

        confirmClose = true;

        return panel?.show();
    }

    export function hide() {
        return panel?.hide();
    }

    function load(model) {
        admin = model?.clone ? model.clone() : new Admin();
        reset(); // reset form
    }

    function reset() {
        changePasswordToggle = false;
        email = admin?.email || "";
        avatar = admin?.avatar || 0;
        password = "";
        passwordConfirm = "";
        setErrors({}); // reset errors
    }

    function save() {
        if (isSaving || !hasChanges) {
            return;
        }

        isSaving = true;

        const data = { email, avatar };
        if (admin.isNew || changePasswordToggle) {
            data["password"] = password;
            data["passwordConfirm"] = passwordConfirm;
        }

        let request;
        if (admin.isNew) {
            request = ApiClient.admins.create(data);
        } else {
            request = ApiClient.admins.update(admin.id, data);
        }

        request
            .then(async (result) => {
                confirmClose = false;
                hide();
                addSuccessToast(admin.isNew ? "Successfully created admin." : "Successfully updated admin.");
                dispatch("save", result);

                if (ApiClient.authStore.model?.id === result.id) {
                    ApiClient.authStore.save(ApiClient.authStore.token, result);
                }
            })
            .catch((err) => {
                ApiClient.errorResponseHandler(err);
            })
            .finally(() => {
                isSaving = false;
            });
    }

    function deleteConfirm() {
        if (!admin?.id) {
            return; // nothing to delete
        }

        confirm(`Do you really want to delete the selected admin?`, () => {
            return ApiClient.admins
                .delete(admin.id)
                .then(() => {
                    confirmClose = false;
                    hide();
                    addSuccessToast("Successfully deleted admin.");
                    dispatch("delete", admin);
                })
                .catch((err) => {
                    ApiClient.errorResponseHandler(err);
                });
        });
    }
</script>

<OverlayPanel
    bind:this={panel}
    popup
    class="admin-panel"
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
            {admin.isNew ? "New admin" : "Edit admin"}
        </h4>
    </svelte:fragment>

    <form id={formId} class="grid" autocomplete="off" on:submit|preventDefault={save}>
        {#if !admin.isNew}
            <Field class="form-field readonly" name="id" let:uniqueId>
                <label for={uniqueId}>
                    <i class={CommonHelper.getFieldTypeIcon("primary")} />
                    <span class="txt">id</span>
                </label>
                <div class="form-field-addon">
                    <i
                        class="ri-calendar-event-line txt-disabled"
                        use:tooltip={{
                            text: `Created: ${admin.created}\nUpdated: ${admin.updated}`,
                            position: "left",
                        }}
                    />
                </div>
                <input type="text" id={uniqueId} value={admin.id} readonly />
            </Field>
        {/if}

        <div class="content">
            <p class="section-title">Avatar</p>
            <div class="flex flex-gap-xs flex-wrap">
                {#each [0, 1, 2, 3, 4, 5, 6, 7, 8, 9] as index}
                    <button
                        type="button"
                        class="link-fade thumb thumb-circle {index == avatar ? 'thumb-active' : 'thumb-sm'}"
                        on:click={() => (avatar = index)}
                    >
                        <img
                            src="{import.meta.env.BASE_URL}images/avatars/avatar{index}.svg"
                            alt="Avatar {index}"
                        />
                    </button>
                {/each}
            </div>
        </div>

        <Field class="form-field required" name="email" let:uniqueId>
            <label for={uniqueId}>
                <i class={CommonHelper.getFieldTypeIcon("email")} />
                <span class="txt">Email</span>
            </label>
            <input type="email" autocomplete="off" id={uniqueId} required bind:value={email} />
        </Field>

        {#if !admin.isNew}
            <Field class="form-field form-field-toggle" let:uniqueId>
                <input type="checkbox" id={uniqueId} bind:checked={changePasswordToggle} />
                <label for={uniqueId}>Change password</label>
            </Field>
        {/if}

        {#if admin.isNew || changePasswordToggle}
            <div class="col-12">
                <div class="grid" transition:slide|local={{ duration: 150 }}>
                    <div class="col-sm-6">
                        <Field class="form-field required" name="password" let:uniqueId>
                            <label for={uniqueId}>
                                <i class="ri-lock-line" />
                                <span class="txt">Password</span>
                            </label>
                            <input
                                type="password"
                                autocomplete="new-password"
                                id={uniqueId}
                                required
                                bind:value={password}
                            />
                        </Field>
                    </div>
                    <div class="col-sm-6">
                        <Field class="form-field required" name="passwordConfirm" let:uniqueId>
                            <label for={uniqueId}>
                                <i class="ri-lock-line" />
                                <span class="txt">Password confirm</span>
                            </label>
                            <input
                                type="password"
                                autocomplete="new-password"
                                id={uniqueId}
                                required
                                bind:value={passwordConfirm}
                            />
                        </Field>
                    </div>
                </div>
            </div>
        {/if}
    </form>

    <svelte:fragment slot="footer">
        {#if !admin.isNew}
            <button type="button" aria-label="More" class="btn btn-sm btn-circle btn-transparent">
                <!-- empty span for alignment -->
                <span />
                <i class="ri-more-line" />
                <Toggler class="dropdown dropdown-upside dropdown-left dropdown-nowrap">
                    <button type="button" class="dropdown-item txt-danger" on:click={() => deleteConfirm()}>
                        <i class="ri-delete-bin-7-line" />
                        <span class="txt">Delete</span>
                    </button>
                </Toggler>
            </button>
            <div class="flex-fill" />
        {/if}

        <button type="button" class="btn btn-transparent" disabled={isSaving} on:click={() => hide()}>
            <span class="txt">Cancel</span>
        </button>
        <button
            type="submit"
            form={formId}
            class="btn btn-expanded"
            class:btn-loading={isSaving}
            disabled={!hasChanges || isSaving}
        >
            <span class="txt">{admin.isNew ? "Create" : "Save changes"}</span>
        </button>
    </svelte:fragment>
</OverlayPanel>
