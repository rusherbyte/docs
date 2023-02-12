<!--
  _____   ____    _   _  ____ _______   ______ _____ _____ _______ 
  |  __  / __   |  | |/ __ __   __| |  ____|  __ _   _|__   __|
  | |  | | |  | | |  | | |  | | | |    | |__  | |  | || |    | |   
  | |  | | |  | | | . ` | |  | | | |    |  __| | |  | || |    | |   
  | |__| | |__| | | |  | |__| | | |    | |____| |__| || |_   | |   
  |_____/ ____/  |_| _|____/  |_|    |______|_____/_____|  |_|   
  This file is auto-generated by script/generate_graphql_api_content.sh,
  please build the schema.json by running `rails api:graph:export`
  with https://github.com/buildkite/buildkite/,
  replace the content in data/graphql_data_schema.json
  and run the generation script `./scripts/generate-graphql-api-content.sh`.
-->
<!-- vale off -->
<h1 class="has-pills" data-algolia-exclude>
  RepositoryProviderGithubEnterprise
  <span class="pill pill--object pill--normal-case pill--large"><code>OBJECT</code></span>
</h1>
<!-- vale on -->


<p>A pipeline's repository is being provided by GitHub Enterprise</p>


{:notoc}

<table class="responsive-table responsive-table--single-column-rows">
  <thead>
    <th>
      <h2 data-algolia-exclude>Fields</h2>
    </th>
  </thead>
  <tbody>
    <tr><td><h3 class="is-small has-pills"><code>name</code><a href="/docs/apis/graphql/schemas/scalar/string" class="pill pill--scalar pill--normal-case pill--medium" title="Go to SCALAR String"><code>String</code></a></h3><p>The name of the provider</p></td></tr><tr><td><h3 class="is-small has-pills"><code>url</code><a href="/docs/apis/graphql/schemas/scalar/string" class="pill pill--scalar pill--normal-case pill--medium" title="Go to SCALAR String"><code>String</code></a></h3><p>This URL to the provider’s web interface</p></td></tr><tr><td><h3 class="is-small has-pills"><code>webhookUrl</code><a href="/docs/apis/graphql/schemas/scalar/string" class="pill pill--scalar pill--normal-case pill--medium" title="Go to SCALAR String"><code>String</code></a></h3><p>The URL to use when setting up webhooks from the provider to trigger Buildkite builds</p></td></tr>
  </tbody>
</table>




<h2 data-algolia-exclude>Interfaces</h2>
<a href="/docs/apis/graphql/schemas/interface/repositoryprovider" class="pill pill--interface pill--normal-case pill--large" title="Go to INTERFACE RepositoryProvider"><code>RepositoryProvider</code></a>