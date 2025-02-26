<script>
  import { getPingData } from "../state/api";
  import {
    pageState,
    updateURLState,
    updateBreadcrumbs,
  } from "../state/stores";
  import { getBigQueryURL } from "../state/urls";

  import VariantSelector from "../components/VariantSelector.svelte";
  import AuthenticatedLink from "../components/AuthenticatedLink.svelte";
  import Commentary from "../components/Commentary.svelte";
  import HelpHoverable from "../components/HelpHoverable.svelte";
  import ItemList from "../components/ItemList.svelte";
  import Label from "../components/Label.svelte";
  import MetadataTable from "../components/MetadataTable.svelte";
  import NotFound from "../components/NotFound.svelte";
  import Markdown from "../components/Markdown.svelte";
  import PageHeader from "../components/PageHeader.svelte";
  import SubHeading from "../components/SubHeading.svelte";
  import AppAlert from "../components/AppAlert.svelte";
  import {
    getLibraryDescription,
    getRecentlyAddedItemDescription,
  } from "../data/help";
  import { PING_SCHEMA } from "../data/schemas";
  import { getLibraryName } from "../formatters/library";
  import { stripLinks } from "../formatters/markdown";
  import { getMetricSearchURL } from "../state/urls";
  import { isRecent } from "../state/items";
  import { getAppBreadcrumbs } from "./AppDetail.svelte";

  export let params;

  let selectedAppVariant;
  const pingDataPromise = getPingData(params.app, params.ping).then(
    (pingData) => {
      updateBreadcrumbs([
        ...getAppBreadcrumbs(params, pingData),
        {
          url: `/apps/${params.app}/pings/${params.ping}/`,
          name: pingData.name,
        },
      ]);

      [selectedAppVariant] = $pageState.channel
        ? pingData.variants.filter((app) => app.id === $pageState.channel)
        : pingData.variants;
      return pingData;
    }
  );
</script>

{#await pingDataPromise then ping}
  <PageHeader title={ping.name}>
    <svelte:fragment slot="tags">
      {#if ping.origin && ping.origin !== params.app}
        <a href={getMetricSearchURL(params.app, ping.origin)}
          ><Label
            text={getLibraryName(ping)}
            description={getLibraryDescription("ping", ping.origin)}
          /></a
        >
      {/if}
      {#each ping.tags as tag}
        <a href={getMetricSearchURL(params.app, tag.name)}
          ><Label
            text={tag.name}
            description={stripLinks(tag.description)}
          /></a
        >
      {/each}
    </svelte:fragment>
  </PageHeader>
  <Markdown text={ping.description} inline={false} />

  <SubHeading
    title={"Metadata"}
    helpText={"Metadata about this ping, as defined by the implementor."}
  />
  <MetadataTable appName={params.app} item={ping} schema={PING_SCHEMA} />

  <SubHeading
    title={"Commentary"}
    helpText={"Reviewed commentary from Mozilla data practitioners on this ping."}
  />
  <Commentary item={ping} itemType={"ping"} />

  <SubHeading
    title={"Access"}
    helpText={"Ways to access this metric in Mozilla's data warehouse."}
  />
  {#if isRecent(ping)}
    <AppAlert
      status="warning"
      message={getRecentlyAddedItemDescription(ping.variants.length, "ping")}
    />{/if}
  {#if ping.variants.length > 1}
    <VariantSelector
      name={"app_id"}
      label={"Application Variant"}
      bind:selectedVariant={selectedAppVariant}
      on:change={() => updateURLState({ channel: selectedAppVariant.id })}
      variants={ping.variants}
    />
  {/if}

  {#if selectedAppVariant}
    <table>
      <col />
      <col />
      <tr>
        <td>
          BigQuery
          <HelpHoverable
            content={"The BigQuery representation of this ping."}
            link={"https://docs.telemetry.mozilla.org/cookbooks/accessing_glean_data.html"}
          />
        </td>
        <td>
          <a
            href={getBigQueryURL(
              params.app,
              selectedAppVariant.id,
              params.ping
            )}
          >
            {selectedAppVariant.table}
          </a>
        </td>
      </tr>
      {#if selectedAppVariant.looker_explore}
        <tr>
          <td>
            Looker
            <HelpHoverable
              content={"Explore this ping in Mozilla's instance of Looker."}
            />
          </td>
          <td>
            <AuthenticatedLink href={selectedAppVariant.looker_explore.url}>
              {selectedAppVariant.looker_explore.name}
            </AuthenticatedLink>
          </td>
        </tr>
      {/if}
    </table>
  {/if}

  <SubHeading
    title={"Metrics"}
    helpText={"Metrics that are sent inside this ping."}
  />
  <ItemList
    itemType="metrics"
    items={ping.metrics}
    appName={params.app}
    tagDescriptions={ping.tag_descriptions}
  />
{:catch}
  <NotFound pageName={params.ping} itemType="ping" />
{/await}

<style lang="scss">
  @import "../main.scss";

  @include metadata-table;
</style>
