<template>
  <Head>
    <link rel="apple-touch-icon" sizes="180x180" href="/icons/apple-touch-icon.png">
    <link rel="icon" type="image/png" sizes="32x32" href="/icons/favicon-32x32.png">
    <link rel="icon" type="image/png" sizes="16x16" href="/icons/favicon-16x16.png">
    <link rel="manifest" href="/icons/site.webmanifest">
    <link rel="mask-icon" href="/icons/safari-pinned-tab.svg" color="#5bbad5">
    <link rel="shortcut icon" href="/icons/favicon.ico">
    <meta name="msapplication-TileColor" content="#da532c">
    <meta name="msapplication-config" content="/icons/browserconfig.xml">
    <meta name="theme-color" content="#ffffff">
    <meta name="description" content="Easily manage your electronic parts for your projects.">
  </Head>
  <AppContainer>
    <NuxtLoadingIndicator />
    <AppHeader />
    <PageContainer>
      <slot />
    </PageContainer>
    <AppFooter />
  </AppContainer>
  <UNotifications />
</template>

<script lang="ts" setup>

</script>

<style></style>