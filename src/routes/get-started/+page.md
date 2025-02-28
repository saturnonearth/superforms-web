<script lang="ts">
	import Form from './Form.svelte'
	import SuperDebug from 'sveltekit-superforms/client/SuperDebug.svelte'

	export let data;
	
	let formComponent
	$: form = formComponent && formComponent.formData()
</script>

<svelte:head>

<title>Get started - Tutorial for Superforms</title>
</svelte:head>

## Installation

Install Superforms with `npm` or `pnpm`:

```
npm i -D sveltekit-superforms zod
```

```
pnpm i -D sveltekit-superforms zod
```

## Following along

The easiest way is to open [the Stackblitz project](https://stackblitz.com/edit/sveltekit-superforms-tutorial?file=src%2Froutes%2F%2Bpage.server.ts,src%2Froutes%2F%2Bpage.svelte) for this tutorial.

Otherwise, you can create a new SvelteKit project with `npm create svelte@latest` and copy/paste the code as you go along, or add it to an existing project.

## Creating a Superform

Let's gradually build up a Superform containing a name and an email address.

### Creating a validation schema

The main thing required to create a Superform is a Zod validation schema. It has a quite simple syntax:

```ts
import { z } from 'zod';

// Name has a default value just to display something in the form.
const schema = z.object({
  name: z.string().default('Hello world!'),
  email: z.string().email()
});
```

This schema represents the form data. It should always start with `z.object({ ... })`, encapsulating a single form.

The [Zod documentation](https://zod.dev/?id=primitives) has all the details for creating schemas, but this is all you need to know for now.

### Initializing the form in the load function

Let's use this schema with Superforms on the start page of the site:

**src/routes/+page.server.ts**

```ts
import { z } from 'zod';
import { superValidate } from 'sveltekit-superforms/server';
import type { PageServerLoad } from './$types';

const schema = z.object({
  name: z.string().default('Hello world!'),
  email: z.string().email()
});

export const load = (async (event) => {
  // Server API:
  const form = await superValidate(event, schema);

  // Always return { form } in load and form actions.
  return { form };
}) satisfies PageServerLoad;
```

The Superform server API is called `superValidate`. Its first parameter is the data that should be sent to the form. It can come from:

- `FormData` in a POST request
- From a database call in the load function, so the form will be populated directly (very useful for backend interfaces)
- Or it can be empty, in which case it will be filled with default values based on the schema. `z.string()` results in an empty string, for example.

In our case, the form should be initially empty, so we can use the `RequestEvent`, which then looks for `FormData`, but as the load function is a `GET` request, it won't find any and will return the default values for the schema.

Note that at the end of the load function we return `{ form }`. As a rule, you should always return the validation object to the client in this manner.

### Displaying the form

Now when we have sent the validation data to the client, we will retrieve it using the client part of the API:

**src/routes/+page.svelte**

```svelte
<script lang="ts">
  import type { PageData } from './$types';
  import { superForm } from 'sveltekit-superforms/client';

  export let data: PageData;

  // Client API:
  const { form } = superForm(data.form);
</script>

<form method="POST">
  <label for="name">Name</label>
  <input type="text" name="name" bind:value={$form.name} />

  <label for="email">E-mail</label>
  <input type="email" name="email" bind:value={$form.email} />

  <div><button>Submit</button></div>
</form>
```

`superForm` is used on the client to display the data, conveniently supplied from `data.form`.

This is what the form should look like now:

<Form {data} bind:this={formComponent} />

### Debugging

Now we can see that the form is populated. But to get deeper insight, let's add the Superform Debugging Svelte Component called `SuperDebug`:

**src/routes/+page.svelte**

```svelte
<script lang="ts">
  import SuperDebug from 'sveltekit-superforms/client/SuperDebug.svelte';
</script>

<SuperDebug data={$form} />
```

This should be displayed:

<SuperDebug data={$form} />

Edit the form fields and see how the data is automatically updated. The component also displays the current page status in the right corner.

### Posting data

Let's add a minimal form action, to be able to post the data back to the server:

**src/routes/+page.server.ts**

```ts
import type { Actions, PageServerLoad } from './$types';
import { fail } from '@sveltejs/kit';
import { superValidate } from 'sveltekit-superforms/server';

export const actions = {
  default: async (event) => {
    // Same syntax as in the load function
    const form = await superValidate(event, schema);
    console.log('POST', form);

    // Convenient validation check:
    if (!form.valid) {
      // Again, always return { form } and things will just work.
      return fail(400, { form });
    }

    // TODO: Do something with the validated data

    // Yep, return { form } here too
    return { form };
  }
} satisfies Actions;
```

Submit the form, and see what's happening on the server:

```ts
POST {
  valid: false,
  errors: { email: [ 'Invalid email' ] },
  data: { name: 'Hello world!', email: '' },
  empty: false,
  message: undefined,
  constraints: {
    name: { required: true },
    email: { required: true }
  }
}
```

This is the validation object returned from `superValidate`, containing all you need to handle the rest of the logic:

- `valid` - Tells you whether the validation succeeded or not.
- `errors` - An object with all validation errors.
- `data` - The posted data, in this case not valid, so it should be returned to the client using `fail`.
- `empty` - Tells you if the data passed to `superValidate` was empty, as it was in the load function.
- `message` - A property that can be set as a general information message.
- `constraints` - An object with [html validation constraints](https://developer.mozilla.org/en-US/docs/Learn/Forms/Form_validation#using_built-in_form_validation) that can be spread on input fields.

### Displaying errors

Now we know that validation has failed, and there are some errors being sent to the client. So how do we display them?

We do that by adding properties to the destructuring assignment of `superForm`:

**src/routes/+page.svelte**

```svelte
<script lang="ts">
  const { form, errors, constraints } = superForm(data.form);
  //            ^^^^^^  ^^^^^^^^^^^
</script>

<form method="POST">
  <label for="name">Name</label>
  <input
    type="text"
    name="name"
    data-invalid={$errors.name}
    bind:value={$form.name}
    {...$constraints.name}
  />
  {#if $errors.name}<span class="invalid">{$errors.name}</span>{/if}

  <label for="email">E-mail</label>
  <input
    type="email"
    name="email"
    data-invalid={$errors.email}
    bind:value={$form.email}
    {...$constraints.email}
  />
  {#if $errors.email}<span class="invalid">{$errors.email}</span>{/if}

  <div><button>Submit</button></div>
</form>

<style>
  .invalid {
    color: red;
  }
</style>
```

As you see, by including `errors` we can display errors where it's appropriate, and through `constraints` we get browser validation even without javascript enabled.

We now have a fully working form with convenient handling of data and validation both on client and server!

There are no hidden DOM manipulations or other secrets, it's just html attributes and Svelte stores.

## Next steps

This concludes the tutorial, but you'd probably want to enable client-side functionality, to take full advantage of the features and enhancements that Superforms bring. 

To do that, take a look at [use:enhance](/concepts/enhance) under the Concepts section in the navigation. All pages here contain interactive examples that helps you use the library to its fullest.

When you've gone over all the concepts, check the [API reference](/api) for a full list of properties returned from `superForm`, and all options that you can use.

If you're ready for a more advanced tutorial, check out [Designing a CRUD interface](/crud), which shows how to make a fully working backend in about 150 lines of code.
