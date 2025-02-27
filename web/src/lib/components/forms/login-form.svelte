<script lang="ts">
  import { goto } from '$app/navigation';
  import LoadingSpinner from '$lib/components/shared-components/loading-spinner.svelte';
  import { AppRoute } from '$lib/constants';
  import { featureFlags, serverConfig } from '$lib/stores/server-config.store';
  import { oauth } from '$lib/utils';
  import { getServerErrorMessage, handleError } from '$lib/utils/handle-error';
  import { getServerConfig, login } from '@immich/sdk';
  import { onMount } from 'svelte';
  import { fade } from 'svelte/transition';
  import Button from '../elements/buttons/button.svelte';
  import PasswordField from '../shared-components/password-field.svelte';

  export let onSuccess: () => unknown | Promise<unknown>;
  export let onFirstLogin: () => unknown | Promise<unknown>;
  export let onOnboarding: () => unknown | Promise<unknown>;

  let errorMessage: string;
  let email = '';
  let password = '';
  let oauthError = '';
  let loading = false;
  let oauthLoading = true;

  onMount(async () => {
    if (!$featureFlags.oauth) {
      oauthLoading = false;
      return;
    }

    if (oauth.isCallback(window.location)) {
      try {
        await oauth.login(window.location);
        await onSuccess();
        return;
      } catch (error) {
        console.error('Error [login-form] [oauth.callback]', error);
        oauthError = (await getServerErrorMessage(error)) || 'Unable to complete OAuth login';
        oauthLoading = false;
      }
    }

    try {
      if ($featureFlags.oauthAutoLaunch && !oauth.isAutoLaunchDisabled(window.location)) {
        await goto(`${AppRoute.AUTH_LOGIN}?autoLaunch=0`, { replaceState: true });
        await oauth.authorize(window.location);
        return;
      }
    } catch (error) {
      handleError(error, 'Unable to connect!');
    }

    oauthLoading = false;
  });

  const handleLogin = async () => {
    try {
      errorMessage = '';
      loading = true;

      const user = await login({ loginCredentialDto: { email, password } });
      const serverConfig = await getServerConfig();

      if (user.isAdmin && !serverConfig.isOnboarded) {
        await onOnboarding();
        return;
      }

      if (!user.isAdmin && user.shouldChangePassword) {
        await onFirstLogin();
        return;
      }
      await onSuccess();
      return;
    } catch (error) {
      errorMessage = (await getServerErrorMessage(error)) || 'Incorrect email or password';
      loading = false;
      return;
    }
  };

  const handleOAuthLogin = async () => {
    oauthLoading = true;
    oauthError = '';
    const success = await oauth.authorize(window.location);
    if (!success) {
      oauthLoading = false;
      oauthError = 'Unable to login with OAuth';
    }
  };
</script>

{#if !oauthLoading && $featureFlags.passwordLogin}
  <form on:submit|preventDefault={handleLogin} class="mt-5 flex flex-col gap-5">
    {#if errorMessage}
      <p class="text-red-400" transition:fade>
        {errorMessage}
      </p>
    {/if}

    <div class="flex flex-col gap-2">
      <label class="immich-form-label" for="email">Email</label>
      <input
        class="immich-form-input"
        id="email"
        name="email"
        type="email"
        autocomplete="email"
        bind:value={email}
        required
      />
    </div>

    <div class="flex flex-col gap-2">
      <label class="immich-form-label" for="password">Password</label>
      <PasswordField id="password" bind:password autocomplete="current-password" />
    </div>

    <div class="my-5 flex w-full">
      <Button type="submit" size="lg" fullwidth disabled={loading}>
        {#if loading}
          <span class="h-6">
            <LoadingSpinner />
          </span>
        {:else}
          Login
        {/if}
      </Button>
    </div>
  </form>
{/if}

{#if $featureFlags.oauth}
  {#if $featureFlags.passwordLogin}
    <div class="inline-flex w-full items-center justify-center">
      <hr class="my-4 h-px w-3/4 border-0 bg-gray-200 dark:bg-gray-600" />
      <span
        class="absolute left-1/2 -translate-x-1/2 bg-white px-3 font-medium text-gray-900 dark:bg-immich-dark-gray dark:text-white"
      >
        or
      </span>
    </div>
  {/if}
  <div class="my-5 flex flex-col gap-5">
    {#if oauthError}
      <p class="text-center text-red-400" transition:fade>{oauthError}</p>
    {/if}
    <Button
      type="button"
      disabled={loading || oauthLoading}
      size="lg"
      fullwidth
      color={$featureFlags.passwordLogin ? 'secondary' : 'primary'}
      on:click={handleOAuthLogin}
    >
      {#if oauthLoading}
        <span class="h-6">
          <LoadingSpinner />
        </span>
      {:else}
        {$serverConfig.oauthButtonText}
      {/if}
    </Button>
  </div>
{/if}

{#if !$featureFlags.passwordLogin && !$featureFlags.oauth}
  <p class="p-4 text-center dark:text-immich-dark-fg">Login has been disabled.</p>
{/if}
