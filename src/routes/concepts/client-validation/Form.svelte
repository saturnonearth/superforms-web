<script lang="ts">
  import type { PageData } from './$types';
  import { superForm } from 'sveltekit-superforms/client';
  import { page } from '$app/stores';
  import Debug from '$lib/Debug.svelte';
  import { tick } from 'svelte';

  export function formData() {
    return form;
  }

  let newTag = '';
  let newTagEl: HTMLInputElement;

  export let data: PageData;

  const { form, errors, enhance, message, constraints } = superForm(data.form, {
    validators: {
      tags: (tag) =>
        tag.length < 2 ? 'Tag must be at least 2 characters' : null
    },
    defaultValidator: 'clear'
  });

  async function addTag() {
    if (!newTag) return;
    $form.tags = [...$form.tags, newTag];
    await tick();
    setTimeout(() => (newTag = ''), 1);
  }
</script>

<Debug open={true} data={$form} />

<form
  method="POST"
  action={$page.url.pathname}
  class="p-5 border-dashed bg-slate-900 border-2 border-primary-900 rounded-xl space-y-4"
  use:enhance
>
  {#if $message}
    <h3 class="rounded p-2 bg-green-700">{$message}</h3>
  {/if}
  <label for="tags" class="label">
    <span>Tags</span>
  </label>
  {#each $form.tags as _, i}
    <input
      class="input"
      type="text"
      name="tags"
      bind:value={$form.tags[i]}
      data-invalid={$errors.tags?.[i]}
    />
    {#if $errors.tags?.[i]}<span class="text-red-500">{$errors.tags[i]}</span
      >{/if}
  {/each}

  <input
    class="input"
    type="text"
    placeholder="Add new tag..."
    bind:value={newTag}
    bind:this={newTagEl}
    on:change={() => addTag()}
  />

  {#if newTag}
    <input
      class="input"
      type="text"
      placeholder="Add new tag..."
      on:focus={() => newTagEl.focus()}
    />
  {/if}

  <div>
    <button type="submit" class="btn variant-filled">Submit</button>
  </div>
</form>
