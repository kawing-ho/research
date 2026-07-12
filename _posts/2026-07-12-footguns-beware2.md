---
layout: post
date: 2026-07-12
title: "Footguns Beware 2: Auth0 + Next.js"
description: Mess with the props, your tokens get popped!
pin: false
mermaid: true
categories: [Research]
tags: [auth0,nextjs]
image:
  path: /assets/images/footgun2-bike.png
  alt: mess with the props, your tokens get popped!
---

## Background

Fresh off of a recent test, I found another interesting footgun which I thought I'd share. It is somewhat similar to the last post where server-side information gets exposed on the client-side but the nature of it is completely different!  

This time the behavior is easily reproducible, but is not very widespread -- nevertheless I hope you enjoy the read and can takeaway something useful from this blog!

> Note: Frameworks are huge beasts that cannot be fully explained in a single blog post, so I have only included the information relevant to the footgun in this post.

### What is Auth0?

> Auth0 is an identity and access management (IAM) platform that handles user authentication and authorization for your applications. Instead of building and maintaining your own login system, you integrate with Auth0, which manages user sign-in, single-sign on, multi-factor authentication, session management, and token issuance. After a user successfully authenticates, Auth0 returns **signed tokens** or **session information** that your application uses to identify the user and enforce access controls.  _-ChatGPT_

The `appSession` cookie returned to each user upon login is a stateless, encrypted cookie created by the Auth0 Next.js SDK (@auth0/nextjs-auth0) or other server-side Auth0 SDKs to track user authentication locally.

Most importantly, it is marked as HttpOnly, Secure, and uses SameSite=Lax by default; therefore, any JavaScript running in the browser cannot read or modify it.

```
eyJhbGciOiJkaXIiLCJlbmMiOiJBMjU2R0NNIiwiaWF0IjoxNzgzNTE5MDcwLCJ1YXQiOjE3ODM1MTkwNzEsImV4cCI6MTc4MzYwNTQ3MX0..OHizOUTpD-gv4jWe.6SF_5SDlJBTnyENdhQKznPkDYP4LeDw3o-pG3jNoNVQx4Ic0Lw-2InJu4CeYyHzzbMPlT7kl569_EalZOZ-P3BQev_GGE8nBnyFGJ7cd0qRbZTbG9t9Ta0Fo1OJhEiUBWl_gqrBlNq_WGAc3P8163ap8Sw0aDfZ1O-khIVB_ZrdnkQ9nBGas192piG18qaVPSLI4Woa36fj4W_AhtWYJZqaNpLmI7Trjf2LdTC6cNZ0jrZ7yMyH5BI3SuvhJ6IMLFFh_FCd_uJNnP14hP-JP1NMAhd4-BXGBK0jLgw_8oN_ApR0iYR03FcSgyQEDkqkoT8hcun4dMtk5_7mZsPFfjR6uxSXzy9vt7SppN-EMBdXii3Lhq6sSWTnfbLIHS0vntE7Vk_iz1slJ0gvEU_1TWP5ikTZpHlN76QdbCBS4A3y6s5_hITJ3BjJL6tTaYrUMFlEMjzMuVisLIzdvxz7KEmHdZDkdPQHBgh_GzWG7tHEgd3BeVb4gq3o1kSAQGzbjjMupQ6NTT8scjXN4uhzuAoBTg21LlqZQ_E0NqYflTe9YzMIUBPEW56e9gX9qbQj8de4HxU39ok67N0OTEWdCCGEoBPW45VcF25eQgyDQpwcCmTkraD8uYE9tR2FUqfAO9Q017NgMeA8Ejh_31hRuoTvyW6A5mxc_y6W8R3wgi6pKiK-rEefHx-k4X9foROg2V87ZuB7jEh5jKEPMMfLrXFONbc2iCUBrv5hgSv-bV-BcOgCmAWUZ1hJWYg0GiT1XVk45BRvOqBYp4aEnaVfvUjTSlKhADMueVS8wP-fgaEbk13RVu6k13cMTh_9vfuW1hv6nOVDVozalu04JuftjmFX2h0CRN0WCwLva58PlY0-hsXaNmPDaYgWsKhFURLK89fDplAJuKVgEhrPPaWqjtDaRhdN-WAPdF8ho9Ad3MW2YZ-ZhjjYDgARZf5_8ZPSjZawQsMpGi6toBkR-rLLQhN8-FtgvBoHl-MA5SVOsWODdHdAx2xyIPd697ZRooFkf-mjOzjtjbwepeBETmVSvE-XOUj3e0jg8e9C88qS32Z2RLmnrb4EZA1QFXxsBPD13JeiHGaVmPZ2GArU4vnZswQ4s5Migaif1MXEzY-dxQEWFptq9dIbzWEQks2e8BiQj_saGv3xlC08FVz3fvzP9x9KXmbGYGkpbtn3RBHwAy0y-kUvgkEUSqOMXd9iIVCzyXMVlv1HetZ6LNSCdD2dWmyTgaNlNDAzdJMjtJCDo4EtSpHx-_YMo7OPmjJRztkguv3lvGDQNID_t8PvW6zoypcEJu3ZAb2Q7MW8-V4oqOurerTL5X6WXu5i4YjPzsbYDM-upvr4WUqO_KlHNxIFzLL9RjYtQ63JKtkaUZY6uaSBMgvAE1k-fgUxDF730ASr9tpKICbR-O_Le-Z_0UY3eg14gBuVH3zjGk_A_iibpd4JVCHd9SoDoovT91t6T823ZHuwxDy4dTRmiZPEoqB_5Lr5zwv9anFjv6veMQNzpEy0E3z7rjbf0oifM2aa7BwmBoDw83g1D5x_VBmxCtS9YtP6PD7KBce-pXVypKITc_nLXuWaLBMIzBcxcxcdnAEezOaieTcoGKZNr-KR0ZbuGKCh6f2WAYNnKC3ESIqgn7ktbk4hhSFbO7AOuiONKFlJ2McPDUgnW4jfrfwJ0hdHGuzVm_fcryajZH5UMq88jNATNmRj2yfgWT9ZwnkZOHa99zb6RA_iCHyDhccFQSxXJ0iaExfrnuxy3fJ3fMKjlG7RjZfHWY53ayTBmxQYxKEKW0zrYJRYFVfAtO4rlx_ufvHt_GmVa2-zQS81GFnb7X3VTMXwRRCtGqd-sYjF6ZMlLhx8P2fOtpPXVEm4psGt1mG54MOhrSJ-ABxTl3oL8CfGTd7mPXOkou-QondlKh2usJlkD43wdzn9b4TiumgvXlk4h6llX3Rd2jvYFlemjoUJmIKOlI3OJ4FrJDlPfveOBbB7b6v4xzCjkzDifIHo-9mLMulCe2K1x_j7871_b2dc7KOEjnxj424wFXTXRzT_i6lFDC85Nr4ddrZ75Iow9WZLvuTPDaZJPZVwS_Q07Q1U01ITQo3FfDy25pVImj5TArj9oR3V8MFReK18LoKIRNghylN3Nx0_eMgDXVpLAENHf5VeoRFPSwNtGT_NKBmw7d-LeSjuy1MJ78LOe3WGkklX2OWMFIr-r5sdao1Ikz6H3HBb1E9Ya_GryL2ar6MW59O6uqTwdO3rPb_vjvIYzz3GTxWtwK_p4sCGDFJYUgTGUgmyiwH1BW-pPsVRN0iemeeZ4ICeZ-5FoCR2fqW4Hm039ITkQExv0X45UO3mqbqI4_kWonLcAVXX5F8SeSlFJ3XbHTlP1rPybC2iLA4_Wu3sPM0uqdW97u3Fo8ohYfa7bJomDYborHaBdkL2YIMgJ1ji3A8GqqsEWfF2DyYO225MFV-Qq6BBzJMmgpbj6VHF5w7n6VnVErh1kED4bPX1_9GnZUt5hB4OfdMQJHcVmB3s_hag7p6XEzAuok7ptstw-MsYAgDeLeE3tXwNsZLdsapWwIb1W9-4pTbihyio_qpoCaeh_fWMd8S2A5ajWTwWX3LrbhVt5hDsDIg7iK7YlZKR1utcknGgVBBPgsnnOIj79nkcHUhb2GC7ZU2LJXhPW3XwTP3Na0R8TtM66suXL6cprw-RW14Z6V-EvpTfr_Nv5LmES9gIU8vwB9nN_httR2NGGPf4glg4_IC_YPOtUdLYwwEXkHAMnngzfzOZdbcOba0lZy7gnAfcF2CNhI68rKkJye5PGPLhoWdkXEzbtE8zob8zrZSGePuEpOrnx6h42pQQN6R2FG4O8FJT6ktc.TIgwFTFh15qFlW0JoxT3hQ
```
{: file='Example of an encrypted appSession cookie'}
  
It stores the user's ID token, access token, refresh token, and profile data right inside the cookie. To prevent decoding or tampering, it is completely encrypted using the `AUTH0_SECRET`.

### What is Next.js?

> Next.js is a React framework for building full-stack applications, providing a structured way to develop both **client-side** and **server-side** functionality. Next.js provides a cohesive framework that handles routing, **rendering**, bundling, and deployment while still allowing you to build applications using React components.  _-ChatGPT_

**Props** are components used to pass values from parent components to child components. In Next.js, it is possible for server-generated data to be passed into the client-side as props too!  

**Hydration** is the process where:

- The server sends HTML to the browser.
- Next.js also sends the serialised props (typically as JSON embedded in the page).
- The browser loads the React JavaScript.
- React uses the same props to recreate the component tree and attach event handlers to the existing HTML.


The **[Pages Router](https://nextjs.org/docs/pages){:target="_blank"}** pattern (now considered legacy to **App Router**) has a file-system based router built on the concept of _pages_. When a file is added to the `pages` directory, it becomes automatically available as a route.

The client's application leveraged this pattern which essentially dictated that:
- The repository structures its pages into a `pages` directory
- There will be a special `_app.xxx` file within this `pages` directory (explained further)
- There will be an exported **[Custom App](https://nextjs.org/docs/pages/building-your-application/routing/custom-app){:target="_blank"}** function which handles all pages


>The `pages/_app.xxx` file is a special file in the **Pages Router** pattern. It acts as the root component that wraps every page in a Next.js application.
>A useful way to think about it is:
>  
>```mermaid
>flowchart TD
>    A[Browser request 🌐] --> B[<b>_app.tsx</b> 🚰]
>    B --> C[Current page <br/><ul><li>index.tsx</li><li>about.tsx</li><li>etc.</li></ul>]
>```
{: .prompt-tip}

A legacy Next.js application using this pattern would hydrate the frontend using the following non-sensitive `pageProps` data by default:
```json
{
  "pageProps": {
    "user": {
      "nickname": "foo",
      "name": "foo@bar.com",
      "picture": "https://s.gravatar.com/avatar/xxxxx.png",
      "updated_at": "2026-07-08T13:57:49.271Z",
      "email": "foo@bar.com",
      "email_verified": false,
      "sub": "auth0|6a4e575d50846d92632b226e",
      "sid": "MbOln9w6ohZYMYKz79_9PQIUinz4J0G4"
    }
  }
}
```  
{: file='pageProps'}  

The Custom App (represented by the example `MyApp` function in the lab) returns the `pageProps` observed above:  
```tsx
import { Session } from '@auth0/nextjs-auth0';
import App, { AppContext, AppProps } from 'next/app'
import { auth0 } from '../lib/auth0';

export default function MyApp({ Component, pageProps }: AppProps) {

    return (
        <Component {...pageProps}/> 
    )
}
//  --- 8< ---
```
{:file='pages/_app.tsx'}  

>The updated flow now looks like:
>  
>```mermaid
>flowchart TD
>    A[Browser request 🌐] --> B[<b>_app.tsx</b> 🚰]
>    B --> |" { Component, pageProps } "| C[Current page <br/><ul><li>index.tsx</li><li>about.tsx</li><li>etc.</li></ul>]
>```
{: .prompt-tip}


The `getInitialProps` [function](https://nextjs.org/docs/pages/api-reference/functions/get-initial-props){:target="_blank"} (also considered legacy) is commonly used in conjunction with this pattern, which:
- Also lives in the same `pages/_app.xxx` file
- Implicitly provides additional props for the Custom App

Example usage of the `getInitialProps` function in a standard `Page`:  
```tsx
import { NextPageContext } from 'next'
 
Page.getInitialProps = async (ctx: NextPageContext) => {
  const res = await fetch('https://api.github.com/repos/vercel/next.js')
  const json = await res.json()          
  return { stars: json.stargazers_count } // ¬  (stars -> stars)
}                                         // |
                                          // v
export default function Page({ stars }: { stars: number }) {
  return stars
}
```
{:file='sample from official docs'}

Going back to our `MyApp` example from before, it would look something like this:
```tsx
import { Session } from '@auth0/nextjs-auth0';
import App, { AppContext, AppProps } from 'next/app'
import { auth0 } from '../lib/auth0';

export default function MyApp({ Component, pageProps }: AppProps) {

    return (
        <Component {...pageProps}/>
    )
}

MyApp.getInitialProps = async (appContext: AppContext) => {
  const appProps = App.getInitialProps(appContext)
  return { ...appProps}
}
```
{:file='Example Custom App usage'}

>The updated hydration flow now looks like:
>  
>```mermaid
>flowchart TD
>    A[Browser request 🌐] --> B[<b>_app.tsx</b> 🚰]
>    B --> |" { Component, pageProps } "| C[Current page <br/><ul><li>index.tsx</li><li>about.tsx</li><li>etc.</li></ul>]
>    B --> D[getInitialProps]
>
>    D -->|appProps| C
>```
{: .prompt-tip}

##  The Footgun

> Regardless of whether your route is static or dynamic, any data returned from `getInitialProps` as props will be able to be examined on the client-side in the initial HTML. This is to allow the page to be hydrated correctly. **Make sure that you don't pass any sensitive information that shouldn't be available on the client in props.**
> - `getInitialProps` [documentation](https://nextjs.org/docs/pages/api-reference/functions/get-initial-props){:target="_blank"}
{: .prompt-danger}


In this case, the client application had a Custom App using `getInitialProps` that:
- loaded the Auth0 session object (containing auth data)
- did some stuff with it
- passed it into the returned props 

This manifested in the Custom App code as follows:
```tsx
//  --- 8< ---
MyApp.getInitialProps = async (appContext: AppContext) => {
  const appProps = App.getInitialProps(appContext)
  console.log(appProps);

  const session =  // #1
    appContext.ctx.req && appContext.ctx.res ?
      await auth0.getSession(appContext.ctx.req, appContext.ctx.res)
    : undefined
    
  return { ...appProps, session } // #2
}
```
{:file='Contrived bad example Custom App'}  
- **#1**: Result of `auth0.getSession` stored in `session` variable
- **#2**: `session` variable passed into returned `props`

>The _footgun_ flow now looks like:
>  
>```mermaid
>flowchart TD
>    A[Browser request 🌐] --> B[<b>_app.tsx</b> 🚰]
>    B --> |" { Component, pageProps } "| C[Current page <br/><ul><li>index.tsx</li><li>about.tsx</li><li>etc.</li></ul>]
>    B --> D[getInitialProps]
>
>    D -->|appProps| C
>    D -->|session | C
>    linkStyle 4 stroke:#d33,stroke-width:3px,color:#d33
>```
{: .prompt-warning}

## Impact

The misused pattern resulted in the Auth0 `session` object serialised into frontend-accessible page data (in this case within the `__NEXT_DATA__` script block) on every page (due to the nature of `_app.tsx`).  
  
> The getSession() method returns a complete `session` object containing the user profile and **all available tokens** (access token, ID token, and refresh token when present) ...
> - `nextjs-auth0` [docs](https://github.com/auth0/nextjs-auth0/blob/main/EXAMPLES.md#on-the-server-app-router){:target="_blank"}

```html
<script src="/_next/static/chunks/react-refresh.js"></script>
<script id="__NEXT_DATA__" type="application/json">
{"props":
{"session":{
"user":{
  "nickname":"ighblfyu3q",
  "name":"ighblfyu3q@gmeenramy.com",
  "picture":"https://s.gravatar.com/avatar/d43c7cfe33114431407a2f9ec5417615?s=480\u0026r=pg\u0026d=https%3A%2F%2Fcdn.auth0.com%2Favatars%2Fig.png",
  "updated_at":"2026-07-12T11:25:49.261Z",
  "email":"ighblfyu3q@gmeenramy.com",
  "email_verified":false,
  "sub":"auth0|6a5379bd04689c37ba1571bb",
  "sid":"R39V5qQEzuEBjtOoxYDCUlXATO9FSV4F"},
// <-- JWT -->
"idToken":"eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjBCSHB5OFYtSno2YUZLNmZaNDlnXyJ9.eyJuaWNrbmFtZSI6ImlnaGJsZnl1M3EiLCJuYW1lIjoiaWdoYmxmeXUzcUBnbWVlbnJhbXkuY29tIiwicGljdHVyZSI6Imh0dHBzOi8vcy5ncmF2YXRhci5jb20vYXZhdGFyL2Q0M2M3Y2ZlMzMxMTQ0MzE0MDdhMmY5ZWM1NDE3NjE1P3M9NDgwJnI9cGcmZD1odHRwcyUzQSUyRiUyRmNkbi5hdXRoMC5jb20lMkZhdmF0YXJzJTJGaWcucG5nIiwidXBkYXRlZF9hdCI6IjIwMjYtMDctMTJUMTE6MjU6NDkuMjYxWiIsImVtYWlsIjoiaWdoYmxmeXUzcUBnbWVlbnJhbXkuY29tIiwiZW1haWxfdmVyaWZpZWQiOmZhbHNlLCJpc3MiOiJodHRwczovL2Rldi1ncTRlN3dsc2tscGNhbTFlLnVzLmF1dGgwLmNvbS8iLCJhdWQiOiIxcWJ4bk5mbHpXN3Z6OVFHYk52OXpUMVoySkRvY3NUMiIsInN1YiI6ImF1dGgwfDZhNTM3OWJkMDQ2ODljMzdiYTE1NzFiYiIsImlhdCI6MTc4Mzg1NTU1MCwiZXhwIjoxNzgzODkxNTUwLCJzaWQiOiJSMzlWNXFRRXp1RUJqdE9veFlEQ1VsWEFUTzlGU1Y0RiIsIm5vbmNlIjoiakg3dmxiN25rWVNTR0hoR1hLaU5VQVh1S3p6RHZSM3hWMFp6TG8tc3hlRSJ9.Wt4BaKktfDc4AGip1ZLxeerjk4w_Ax3bpOYZPix7kHfycQAADQ2NeJ6GdXH-XJK5rOkUJwtoh1cVkIapoByhOPoEdu_NUxMYrasLxHYEhMZcE-4OPP1k6qmK3F3iOG0tLjkZ3241mN4laIVox9pQIIl79B93sAulE-mn5MqHxrhyCkCCc8fREzk5BZNlI1XAFjLtjkt-VmKVDCQEC4psHPd4cjFkUOaQAMuLJ3VLm2suwpz9kAfmvqZ9kHFif8tRzX5CGulQN9QmJLC4xhuJipLGgSXgdXloNZsNsATayDK4Wuh7P9XqlrGIAillSzv-XK3iS24u8T0Ex56YYVZPDQ",
"accessToken":"eyJhbGciOiJkaXIiLCJlbmMiOiJBMjU2R0NNIiwiaXNzIjoiaHR0cHM6Ly9kZXYtZ3E0ZTd3bHNrbHBjYW0xZS51cy5hdXRoMC5jb20vIn0..QsE3D_-vPov7dYix.j_stI6eMxHX2O4YrM2DUS76Gh_yTNYjc1j-pZdseeyUA82Z8AbE7HaJKc8evssZKqYBGIO_WLDF6au2wHvYWwGVaEuXxV3gpbGRhwqn894uWSTNwC1XHmf2hsRoez56eiUYImJhA3l-DWOxv2wAYvkL2HSYPDLjxBATL0bU5DHJcTsL4ZzNda-je1KF1eUY6A3hOIh3ZeD_qKv1MoX3-YCM6lkdavU5ZhXmDkM1D3E1415SUHIaEzrNu_kprx2STrNifohj9vEmVlM-v-m0kr2D8Cy6rINezvB4xbaeI-nm52ncOFM7zb8xi_4iBWilRtWLWaf0bMbY-KO0fuExc726w.7F9OfKmaYBrAYOcGYrabog",
// <-- /JWT -->
"accessTokenScope":"openid profile email",
"accessTokenExpiresAt":1783941950,
"token_type":"Bearer"},
"__N_SSP":true,
"pageProps":{"user":{"nickname":"ighblfyu3q","name":"ighblfyu3q@gmeenramy.com","picture":"https://s.gravatar.com/avatar/d43c7cfe33114431407a2f9ec5417615?s=480\u0026r=pg\u0026d=https%3A%2F%2Fcdn.auth0.com%2Favatars%2Fig.png","updated_at":"2026-07-12T11:25:49.261Z","email":"ighblfyu3q@gmeenramy.com","email_verified":false,"sub":"auth0|6a5379bd04689c37ba1571bb","sid":"R39V5qQEzuEBjtOoxYDCUlXATO9FSV4F"}}},
"page":"/","query":{},"buildId":"development","isFallback":false,"gssp":true,"appGip":true,"scriptLoader":[]}
</script></body></html>
```
{:file='prettified NEXT_DATA block in HTML'}

This practice greatly increases the attack surface for token theft and misuse. The application's use of server-side sessions already provides a mechanism for maintaining user authentication without exposing the raw bearer tokens to the frontend. Exposing raw tokens to the frontend in this way removes important security boundaries and increases the impact of other vulnerabilities.

For example:
- Any Cross-Site Scripting (XSS) vulnerability would allow an attacker to directly read and exfiltrate the access token.
- The exposure violates the principle of least privilege by providing the browser access to credentials that is normally only required by server-side application components.
- The tokens appeared to be valid for 10 hours by default, and 24 hours in the client-configured environment

## How bad is it?

Currently, a sample GitHub [search](https://github.com/search?q=path%3A%2F_app%5C.*%2F+getInitialProps+AND+auth0+AND+%2Fawait.*%5C.getSession%2F&type=code){:target="_blank"} reveals that only ~7 public repos use `getInitialProps` within their `pages/_app.xxx` file.

After looking at all 7 files, only 2 of them (which are dead projects) explicitly pass the session object into the returned props, so it's safe to say this footgun is not that common!

> It should be worth noting that:
> - This is likely because the Pages Router pattern is deprecated in favour of App Router now, so newer projects would avoid this pitfall entirely
> - This case study only covers **Auth0**, this footgun behaviour would likely affect other authentication providers that also integrate with Next.js.
{: .prompt-info}


## Reproduction Steps

#### 1. Sign up for and set up a free basic Auth0 tenant

![regapp](/assets/images/footgun2-regapp.png)
_Select Regular Web Application during Creation of new Application_

![settings](/assets/images/footgun2-settings.png)
_Take note of these values in the Settings tab_

> You may need to adjust the Allowed URIs/URLs section of your app,  
> depending on how you setup the Auth0 you may need to add additional paths
> ![allowed](/assets/images/footgun2-allowed.png)
> _These two are most likely to cause problems during dev-ing time_
> As an example I had the following set as my Allowed Callback URLs:
> > http://localhost:3000/api/auth/callback   
> > http://localhost:3000/auth/callback   
{: .prompt-info}


#### 2. Set the appropriate Auth0 secrets in `.env.local`

```sh
APP_BASE_URL=http://localhost:3000
AUTH0_DOMAIN=foobar.auth0.com
AUTH0_CLIENT_ID=REPLACEME
AUTH0_CLIENT_SECRET=REPLACEME
AUTH0_SECRET=REPLACEME
```
 {: file='.env.local'}

#### 3. Spin up the **Node** environment via Docker

```shell
# Lab URL: https://github.com/kawing-ho/labs/tree/main/nextjs-auth0
git clone https://github.com/kawing-ho/labs.git
mv .env.local labs/nextjs-auth0
cd labs/nextjs-auth0
docker build . -t nextjs-auth0
docker run --rm -p 3000:3000 -v "$(pwd):/app" -v /app/node_modules nextjs-auth0
```

#### 4. Signup / Login to a fake user account

![login](/assets/images/footgun2-index-unauth.png)
_The index page leads to Auth0 signup/login on click_
    
![signupflow](/assets/images/footgun2-signup.png)
_Follow prompts to signup/login which will return to the app upon success_
  
#### 5. Inspect element and observe tokens in HTML
![tokens](/assets/images/footgun2-tokens.png)
_`idToken` present in `__NEXT_DATA__` script block_
  
#### 6. Remove `session` from return value within `_app.tsx`
Modify `pages/_app.tsx` and save changes:
```tsx
  return { ...appProps, session } // remove 'session'

  return { ...appProps } // fixed
```
{:file='pages/_app.tsx'}  
  
#### 7. Reload pages and observe absence of tokens
![fixed](/assets/images/footgun2-fixed.png)
_Observe the `session` object is no longer present in the JSON blob_
  
## Conclusion

The remediation advice given was to:
* Modify `getInitialProps` to return only the minimum user attributes required by the client-side application (such as display name and permissions), omitting raw session data from the serialised page props.
  * Since the `session` object contains various fields, only the necessary subfields can be passed in as props rather than the entire `session` object itself.
* Retain bearer tokens exclusively on the server side, using the existing Auth0 session cookie mechanism for authentication rather than forwarding tokens to the browser.
* Consider migrating away from the `getInitialProps` pattern entirely, as it is a legacy API. Instead, use `getServerSideProps` on individual pages, which provides finer-grained control over what data is serialised and avoids application-wide risk.

Same as previously, I hope this post can help bring more awareness to this potential pitfall. If you would like to quickly check your projects, the associated semgrep [rule](https://github.com/kawing-ho/semgrep-rules){:target="_blank"} targeted against Auth0 + Next.js is ready to use.
