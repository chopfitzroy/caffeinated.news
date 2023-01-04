# Falling in love with fresh and PocketBase 🐿

One of the things that bothers me most about modern web development is the ceremony associated with project bootstrapping.

When as an industry did we all agree on the ridiculous amount of prolifory processes that need to be setup before we can even start writing code.

Don't get me wrong we have come a long way from manually tweaking Webpack configs or fine tuning Rollup builds but we still have a long way to go.

### Enter fresh 🍋

Recently I have been experimenting with the Deno framework [fresh](https://fresh.deno.dev/) and I have got to say it is a breath of _fresh_ air.

How often do we start some sort of dev server i.e `npm run start` then realise we need some third party plugin so we google for it on NPM, then we cancel the dev server `npm i package` restart the dev server (better hope the package has types or you'll need to install those to) then import the package in our code and continue working.

How often do we need to rememeber to run `npm run build` to produce production builds, or `npm run lint` to make sure our code is meeting the teams agreed upon standards?

A lot of this problem can be abstracted (read hidden) away by using a CI/CD solution which is nice but still results in painfully slow build times. Who wants to open a PR only to find out 15 minutes later it can't be merged because of some linting issue.

Fresh, or more appropiately Deno doesn't solve all of these issues but it makes a really good start. The first and arguably most noticable thing about Deno is being able to install packages adhoc and when required by simply importing the package directly from the URL.

The package is then installed on demand without needing to restart the build process, additionally if the package you are looking for doesn't exist as part of the Deno repositories or on `esm.sh` Deno now has [full NPM support](https://deno.com/blog/v1.28).

Next on the list is out of the box TypeScript support, absolutely no additional tooling required, this might not sound like a big deal because most of us have gone through the pain of setting up TypeScript with Node we have now forgotten the struggle, but what if there was no struggle in the first place? This is where Deno really shines for newer developers or developers transitioning from a different stack who just want to get up and running with TypeScript straight away.

Additional to all of the above fresh support comes baked into Deno deploy which includes just in time rendering of views as well as edge deployments by default, this honestly feels like taking a step back in time (in the best possible way) in the sense that you push your code and two seconds later it is running on the server, no build step, no CI/CD, nothing...

### PocketBase the base that does it all 🚀

We've all heard of Firebase, and a lot of us have even explored some of the opensource alternatives like SupaBase or Appwrite and while these are absolutely fantastic projects they have always frustrated me for one simple reason.

Set up.

Not to be disingenuous Appwrite do provide a docker compose file to bootstrap a lot of this process but the very fact that so many different services need to be orchestrated in order to have a running instance is tedious at best.

PocketBase does a way with all of this by shipping a single runnable binary, download the file and run `pocketbase serve` and your up and running.

A big part of what makes this possible is SQLite which PocketBase ships embedded in it's single binary, it then uses WAL mode to write to a `pb_data/` directory at runtime.

Now you may be concerned that SQLite does not scale and in some instances you would certainly be correct however unless your the next Facebook it's going to be a long time before you run into these kind of issues.

Infact SQLite performanes more or less on par with traditional database solutions and usually outperforms traditional solutions when it comes to database read speed.

So what's the catch? Well we've already touched on it a bit but essentially scaling, unlike a lot of modern solutions PocketBase scales vertically as opposed to horiztonally meaning that you essentially have to beef up your single server to meet demand as opposed to spinning up additional nodes.

What's the benefit of this? Well it's supoer _easy_ managing a single server is incredible simple and modern hosting solutions make adding additional resources an absolute breeze.

One final benefit of PocketBase or more accurately SQLite as a database solution is it allows the use of [Litestream](https://litestream.io/) as a database backup solution, this makes database backups fast, easy, and above all else cheap.

### So now that we've got the tools how do we make them play nice? 🤔

Arguably the hardest part of working with fresh and PocketBase is authentication, fortunately PocketBase include some SSR tips as part of their [SDK documentation](https://github.com/pocketbase/js-sdk#ssr-integration).

Now while they don't cover fresh explicitly it is pretty easy to work from their [SvelteKit](https://kit.svelte.dev/) example.

```ts
import { MiddlewareHandlerContext } from "$fresh/server.ts";
import PocketBase from "https://esm.sh/pocketbase@0.9.0"

export type MiddlewareState = Record<'pb', PocketBase>

export async function handler(
  req: Request,
  ctx: MiddlewareHandlerContext<MiddlewareState>,
) {
  // @NOTE
  // - Similar to recommended SvelteKit implementation
  // - https://github.com/pocketbase/js-sdk#ssr-integration
  ctx.state.pb = new PocketBase("https://api.coffeeandcode.app");
  ctx.state.pb.authStore.loadFromCookie(req.headers.get('cookie') || '');

  try {
    if (ctx.state.pb.authStore.isValid) {
      await ctx.state.pb.collection('users').authRefresh();
    }
  } catch (_) {
    ctx.state.pb.authStore.clear();
  }

  // @NOTE
  // - Cannot mutate headers directly
  // - https://github.com/satyarohith/sift/issues/29
  const resp = await ctx.next();
  const headers = new Headers(resp.headers);

  // @TODO
  // - Use cookies methods from the deno `std` lib
  headers.append('set-cookie', ctx.state.pb.authStore.exportToCookie());

  return new Response(resp.body, { ...resp, headers });
}
```

This works if you want to perform auth as part of each server render and do things like redirect the user if the aren't authenticated etc...

It is worth noting that if you want to use a shared `authStore` between the client and the server you will need to set [`httpOnly: false` in `exportToCookie`](https://github.com/pocketbase/js-sdk/issues/69).

Unless I really need server side authentication I tend to use PocketBase primarily on the client for things like authentication, however if you have public collections i.e `products` and they have no authentication requirements there is nothing stopping you from using a PocketBase instance server side to fetch and display these to improve page speeds.

There is really no _best_ answer just different compromises that come with different pros and cons, whatever the case though it pays to make sure you have a firm understanding of what is processed on the server and what is procesed on the client.


