# Next.js boilerplate for decoupled / headless WordPress applications

This is WordPress VIP's [Next.js][nextjs] boilerplate for decoupled WordPress. It is not required for Node.js applications on VIP, but it helps solve many common use cases for decoupled / headless WordPress applications. If you choose to use it, VIP's [decoupled plugin bundle][bundle] must be installed and activated on the WordPress backend. The notes below describe its behavior when run on [WordPress VIP's platform][wpvip].

> ⚠️ This project is under active development. If you are a VIP customer, please let us know if you'd like to use this boilerplate and we can provide additional guidance. Issues and PRs are welcome. 💖

## Features

+ [Next.js][nextjs] 12
+ Fetch data with [Apollo][apollo] and [WPGraphQL][wpgraphql]
+ Seamless previewing
+ Easily map Gutenberg blocks to React components and incorporate your design system
+ Automatic [code generation][code-generation] from GraphQL queries
+ Optional TypeScript support
+ Sitemap and RSS feeds

## Getting started

### Install dependencies

```sh
npm install
```

### Environment variables

Update the following environment variables defined in the `.env` file:

+ `NEXT_PUBLIC_GRAPHQL_ENDPOINT`: The full URL, including protocol, of your WPGraphQL endpoint. You can find it in the WordPress Admin Dashboard > Settings > VIP Decoupled.
+ `NEXT_PUBLIC_SERVER_URL`: The full URL, including protocol, of this Next.js site. This allows things like sitemaps and link routing to be configured correctly.

If your WPGraphQL endpoint is at a non-default location, you will need to provide the `NEXT_PUBLIC_WORDPRESS_ENDPOINT` environment variable. This should be the root URL of your WordPress installation.

If you have additional environment variables, you can add them here.

### Development server

Start a development server, with hot-reloading, at [http://localhost:3000][local].

```sh
npm run dev
```

### Production build

```sh
npm run build
npm start
```

These are the exact same commands that will be executed when your application runs on WordPress VIP. Testing your production build locally is a good step to take when troubleshooting issues in your production site.

Note that the `build` directory has been added to the `.gitignore` file. This avoids a steady buildup of build artifacts in the git repository, but means that your local build will not be pushed to WordPress VIP. Instead, we'll run the build automatically whenever you push up code changes.

## Previewing

Previewing unpublished posts or updates to published posts works out of the box. Simply click the “Preview” button in WordPress and you’ll be redirected to a one-time-use preview link on the Next.js site. You can share your preview link with others; as long as they are logged in to WordPress in the same browser and have permissions to access that content, they will be able to preview it as well.

## Gutenberg / block support

When you query for content (posts, pages, and custom post types), you'll receive the post content as blocks. If the content was written with WordPress's [block editor][gutenberg] (Gutenberg), these blocks will correspond directly with the blocks you see in the editor. The block data you will receive roughly matches the output of WordPress’s [`parse_blocks` function][parse-blocks], with some enhancements. To learn more, you can follow how block data is parsed and resolved in [our extension of WPGraphQL][content-blocks].

Receiving the content as blocks allow you to easily create customizations defining the related component for each `block`. This boilerplate provides a few mappings for basic components like headings, paragraphs, lists, and tables. To check the list with the current components, you can check the default object returned importing `mapBlockNamesToComponents` from `'@/lib/blocks'` or looking for the method content on [this file](https://github.com/Automattic/vip-go-nextjs-skeleton/blob/trunk/lib/blocks.ts).

Here is a simple example of how to override existing relations or create new ones:

```js
function PostContent ( { blocks } ) {
	const components = mapBlockNamesToComponents({
	    'core/paragraph': CustomParagraphComponent
	});

	return (
		<>
			{
				mapBlocksToRenderComponents( { blocks, components } )
			}
		</>
	);
}
```

See the [`PostContent`][post-content] component for a full working example. You will probably need to write additional components and modify the `switch` statement in `PostContent` in order to support all of the block types and custom blocks that you use in your WordPress instance.

If you used WordPress's [classic editor][classic-editor], you will receive a single block representing the HTML content of the entire post. A `ClassicEditorBlock` component is provided to render these blocks.

### Unsupported blocks

When running the development server, in order to help you identify blocks that have not yet been mapped to a React component, this boilerplate will display an "Unsupported block" notice. This notice is suppressed in production and the block is simply ignored.

## Internal link routing

When writing code that links to another page in your Next.js application, you should use Next.js's [`Link` component][next-link] so that the request is routed client-side without a full-round trip to the server.

However, when user-authored blocks contain links, the `innerHTML` is handled by React and you don't have an opportunity to use the `Link` component. To address this, our boilerplate [listens for link clicks][link-listener] and will route them client-side if the link destination is determined to be internal. You can configure which hostnames are considered internal in [`lib/config`][lib-config].

## Data fetching

Next.js is optimized to create performant pages that are statically generated at build time ([`getStaticProps`][nextjs-gsp]) or server-side-rendered at request time ([`getServerSideProps`][nextjs-gssp]). This results in HTML that is cacheable at the edge and immediately crawlable by search engines—both critically important factors in the performance and success of your site.

This boilerplate uses [Apollo][apollo] to query for data using GraphQL, and it is configured and ready to use. Note that Apollo hooks (e.g., `useQuery`) are not compatible with `getStaticProps` or `getServerSideProps`.

### Client-side data fetching

Many Apollo implementations, including Next.js’s official examples, implement a complex, isomorphic approach that bootstraps and hydrates the data from the server-side render into an in-memory cache, where it can be used for client-side requests. We have intentionally avoided this approach because it introduces a large performance penalty and increases the risk that performance degrades even more over time.

Before adding client-side data fetching, examine your typical user flows in detail and consider whether it truly benefits your application and its users. Skipping this complicated step simplifies your configuration, decreases page weight, and usually increases overall performance. If you absolutely need to perform client-side data fetching, an `ApolloProvider` is exported and ready to use in [`graphql-provider`][apollo-provider]. Note that data from the server-side render will not be hydrated into the store.

### Code generation

Our boilerplate has a code generation step that examines the GraphQL queries in `./graphql/queries/`, introspects your GraphQL schema, and generates TypeScript code that can be used to load data via Apollo. See [`LatestContent`][latest-content] for an example of using generated code with `getStaticProps` and [`Post`][post] for an example with `getServerSideProps`.

Having declared types across the entire scope of data fetching chain—queries, responses, and React component props—is incredibly powerful and provides confidence as you build your site. Code generation runs automatically on all GraphQL queries in `./graphql/queries/` whenever you start the development server or build the application. If you need to run it manually, you can use:

```sh
npm run codegen
```

In development, if you make changes or additions to your queries, you will need to restart the development server to see those changes reflected.

## Caching

Responses from Next.js are cached by [VIP's page cache][page-cache] for five minutes by default, [overriding the default behavior of Next.js][cache-config] to help avoid invalidation issues.

As `POST` requests, GraphQL queries are not cached. However, when using static or server-side data loading—which is strongly recommended—these queries are effectively cached by the page cache.

### Redis

If you have Redis deployed alongside your application hosted on VIP Go, you can cache API responses and other data there. An example preconfigured to work on VIP is provided at `./pages/api/books.ts`.

## Custom server

WordPress VIP's platform requires a [healthcheck endpoint][healthcheck] to assist in monitoring. Providing this endpoint correctly requires a [custom server][nextjs-custom-server], which we provide in [`server/index.js`][server-entrypoint]. It is easy to outgrow Next.js's builtin server, so having a custom server (based on [Express 4][express]) available out-of-the-box can be useful.

## Next.js middleware and edge runtime

Next.js offers a way to implement [middleware][middleware], which can be a great way to separate logic from presentation. Additionally, this middleware can target an ["edge runtime"][edge-runtime] on some platforms (for example, Vercel’s "edge functions"). Middleware is supported on VIP, but will not run at the edge. Instead, middleware runs on your origin servers with the rest of your application.

## Serverless functions

"Serverless" functions that live in `/pages/api/` and target Node.js (via JavaScript or TypeScript) are supported on VIP. Just like middleware, they will run on your origin servers.

## Sitemap and syndication (RSS) feeds

`/sitemap.xml` is [proxied to your WordPress backend][sitemap-proxy], where it is fulfilled by the default sitemap provided by WordPress (or whatever is served there, if it is overridden by a plugin). Any path that ends in `/feed` is [*redirected* to your WordPress backend][feed-redirect]. This matches the rewrite rule that exists in WordPress.

## TypeScript

This boilerplate is written in [TypeScript][typescript]. Next.js has [built-in support for TypeScript][nextjs-ts] and processes it automatically in both development and production. If you're already proficient in TypeScript, see [`tsConfig.json`][ts-config] for details.

You don’t need to use TypeScript to use this boilerplate: our `tsConfig.json` is lenient and allows you to write code in either TypeScript or JavaScript.

## Linting

Next.js provides an [ESLint integration][nextjs-eslint], which means you don't need to separately install `eslint` packages. This boilerplate provides an [ESLint config][eslint-config] based on the default Next.js ESLint rules, which can be integrated with most code editors. To run linting manually, use:

```sh
npm run lint
```

Many linting issues can be fixed automatically with:

```sh
npm run lint:fix
```

## Tests

There is support for tests using [Jest][jest]. Some basic unit tests are provided for boilerplate code and carry a `.test.ts` extension. Run tests using:

```sh
npm test
```

## URL imports

URL imports allow you to import packages or images directly from URLs instead of from local disk. At this time, the feature is not stable, introduces security risk, and is not recommended.

## Image optimization

The Next.js `Image` component, [next/image][nextjs-image], is an extension of the HTML `<img />` element, evolved for the modern web. It includes a variety of built-in performance optimizations. Next.js will automatically determine the width and height of your image based on the imported file.

For the API images, the `srcSet` property is automatically defined by the `deviceSizes` and `imageSizes` properties added to the `next.config.js` file. If you need to manually set the `srcSet` for a particular image, you should use the `<img />` HTML tag instead.

## Breaking changes from earlier Next.js versions

- Webpack 4 support has been removed. See the [Webpack 5 upgrade documentation][webpack5] for more information.
- The `target` option has been deprecated. If you are currently using the `target` option set to `serverless`, please read the [documentation on how to leverage the new output][output-file-tracing].
- Next.js `Image` component changed its wrapping element. See the [documentation][image-optimization] for more information.
- The minimum Node.js version has been bumped from `12.0.0` to `12.22.0` which is the first version of Node.js with native ES Modules support.


[apollo]: https://www.apollographql.com
[apollo-provider]: https://github.com/Automattic/vip-go-nextjs-skeleton/blob/725c0695ad603d2ecc8b56ff1c9f1cad95f5fe98/graphql/apollo-provider.tsx
[block-attributes]: https://developer.wordpress.org/block-editor/reference-guides/block-api/block-attributes/
[bundle]: https://github.com/Automattic/vip-decoupled-bundle
[cache-config]: https://github.com/Automattic/vip-go-nextjs-skeleton/blob/725c0695ad603d2ecc8b56ff1c9f1cad95f5fe98/next.config.js#L34-L51
[classic-editor]: https://wordpress.com/support/classic-editor-guide/
[code-generation]: https://www.graphql-code-generator.com
[content-blocks]: https://github.com/Automattic/vip-decoupled-bundle/blob/trunk/blocks/blocks.php
[edge-runtime]: https://nextjs.org/docs/api-reference/edge-runtime
[eslint-config]: https://github.com/Automattic/vip-go-nextjs-skeleton/blob/725c0695ad603d2ecc8b56ff1c9f1cad95f5fe98/.eslintrc
[express]: https://expressjs.com
[feed-redirect]: https://github.com/Automattic/vip-go-nextjs-skeleton/blob/725c0695ad603d2ecc8b56ff1c9f1cad95f5fe98/next.config.js#L95-L100
[gutenberg]: https://developer.wordpress.org/block-editor/
[healthcheck]: https://docs.wpvip.com/technical-references/vip-platform/node-js/#h-requirement-1-exposing-a-health-route
[image-optimization]: https://nextjs.org/docs/basic-features/image-optimization#styling
[jest]: https://jestjs.io
[latest-content]: https://github.com/Automattic/vip-go-nextjs-skeleton/blob/725c0695ad603d2ecc8b56ff1c9f1cad95f5fe98/pages/latest/%5Bcontent_type%5D.tsx
[lib-config]: https://github.com/Automattic/vip-go-nextjs-skeleton/blob/725c0695ad603d2ecc8b56ff1c9f1cad95f5fe98/lib/config.ts
[link-listener]: https://github.com/Automattic/vip-go-nextjs-skeleton/blob/725c0695ad603d2ecc8b56ff1c9f1cad95f5fe98/lib/hooks/useInternalLinkRouting.ts
[local]: http://localhost:3000
[middleware]: https://nextjs.org/docs/middleware
[nextjs]: https://nextjs.org
[nextjs-custom-server]: https://nextjs.org/docs/advanced-features/custom-server
[nextjs-eslint]: https://nextjs.org/docs/basic-features/eslint
[nextjs-image]: https://nextjs.org/docs/api-reference/next/image
[nextjs-gssp]: https://nextjs.org/docs/basic-features/data-fetching#getserversideprops-server-side-rendering
[nextjs-gsp]: https://nextjs.org/docs/basic-features/data-fetching#getstaticprops-static-generation
[nextjs-link]: https://nextjs.org/docs/api-reference/next/link
[nextjs-ts]: https://nextjs.org/docs/basic-features/typescript
[output-file-tracing]: https://nextjs.org/docs/advanced-features/output-file-tracing
[page-cache]: https://docs.wpvip.com/technical-references/caching/page-cache/
[parse-blocks]: https://github.com/WordPress/wordpress-develop/blob/5.8.1/src/wp-includes/blocks.php#L879-L891
[post]: https://github.com/Automattic/vip-go-nextjs-skeleton/blob/725c0695ad603d2ecc8b56ff1c9f1cad95f5fe98/pages/%5B...slug%5D.tsx
[post-content]: https://github.com/Automattic/vip-go-nextjs-skeleton/blob/725c0695ad603d2ecc8b56ff1c9f1cad95f5fe98/components/PostContent/PostContent.tsx
[server-entrypoint]: https://github.com/Automattic/vip-go-nextjs-skeleton/blob/725c0695ad603d2ecc8b56ff1c9f1cad95f5fe98/server/index.js
[sitemap-proxy]: https://github.com/Automattic/vip-go-nextjs-skeleton/blob/725c0695ad603d2ecc8b56ff1c9f1cad95f5fe98/next.config.js#L95-L100
[ts-config]: https://github.com/Automattic/vip-go-nextjs-skeleton/blob/725c0695ad603d2ecc8b56ff1c9f1cad95f5fe98/tsconfig.json
[typescript]: https://www.typescriptlang.org
[webpack5]: https://nextjs.org/docs/messages/webpack5
[wpgraphql]: https://www.wpgraphql.com
[wpvip]: https://wpvip.com
