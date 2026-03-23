---
layout: post
date: 2026-03-18
title: Footguns Beware
description: Vulnerable conditions that are a sum of its parts.
pin: true
categories: [Research]
tags: [vitejs]
image:
  path: /assets/images/footgun-rake.png
  alt: the odds of landing on a rake are slim, but never zero
---

## Background

I  discovered through a previous pentest two vulnerable behaviours which stemmed from developer footguns, which I'll share in depth below.

One related to ViteJS was notoriously hard to reproduce, which led me down a path to reading countless GitHub issues and online articles. This led to a subsequent observation for ViteJS which I have included for brevity. 

The two ViteJS behaviours are not novel discoveries by me, instead I have documented my testing setup and attempts at reproducing the impactful behaviour below.

<div class="divider"></div>

## 1. Capacitor Logging in 'Production'

### What is Capacitor?
The Capacitor framework by Ionic lets one convert an eligible web application into a version that can run on both mobile and desktop environments.

The framework provides logging interfaces for developers to use when catching errors or other information, which becomes interesting once you factor in the conversion process since:
- logging for a web application goes to backend server logs
- logging for a mobile application goes to the _on device_ application logs

###  The Footgun

The logging configuration can be set via the `capacitor.config.(ts|js|json)`{: .filepath} files:
```typescript
const config: CapacitorConfig = { 
	appId: 'com.foo.bar',
	appName: 'Foobar App',
	loggingBehavior: 'production',   // <-----
	server: {
		androidScheme: 'https',
    },
```
{: file='capacitor.config.ts'}

The [documentation](https://capacitorjs.com/docs/config#schema){:target="_blank"} below provides developers with three main options for logging:
<!-- markdownlint-disable-next-line MD040 -->
```
/**  
*  <-->8-->
*  
* 'none' = logs are never produced  
* 'debug' = logs are produced in debug builds but not production builds  
* 'production' = logs are always produced  
*  
* @since 3.0.0  
* @default debug  
*/  
loggingBehavior?: 'none' | 'debug' | 'production';
```
{: file='Capacitor v3 Documentation '}


As a developer, it would be easy to assume that `debug` would produce logs while `production` did not. Unfortunately this is not the case.

> In my test, this misalignment in assumed and actual behaviour caused a user's plaintext username, password, and session cookies to be logged whenever they performed a sign-in to the mobile app.
> ![footgun-logged](/assets/images/footgun-logged.png)
{: .prompt-danger }


Perhaps `full` would be a better-fitting option in order to avoid misinterpretation of logging behaviour.

### Public Discourse 
I haven't actually seen anyone talk about this yet, understandably so because the logging itself isn't immediately bad, it's just risky - with added impact based on what gets logged.

### Reproduction Steps
No need since its pretty straightforward.

### How bad is it?

> Compared to the other two below, the scope of impact for this one is reduced since the blast zone is restricted to local device.

Currently, a sample GitHub [search](https://github.com/search?q=path%3A%2Fcapacitor.config.%28ts%7Cjs%7Cjson%29%24%2F+AND+%2F%5B%27%22%5D%3FloggingBehavior%5B%27%22%5D%3F%3A+*%5B%27%22%5Dproduction%5B%27%22%5D%2F&type=code){:target="_blank"} reveals ~156 public repos using this configuration:

![footgun-capacitor](/assets/images/footgun-capacitor.png)
_In comparison, this [search](https://github.com/search?q=path%3A%2Fcapacitor.config.%28ts%7Cjs%7Cjson%29%24%2F+AND+%2F%5B%27%22%5D%3FloggingBehavior%5B%27%22%5D%3F%3A+*%5B%27%22%5D%28debug%7Cnone%29%5B%27%22%5D%2F&type=code){:target="_blank"}  shows 140 repos using the more secure configurations._

---

## 2. Vite.JS process.env Direct Use

### What is Vite?

Vite is a modern successor to Webpack, known for its speed. In the same tool, you can:
- run a development server that serves frontend content and handles backend processes
- run a build process for frontend assets to bundle them for production use.

If you can see where this is going, this tool handles both frontend and backend stuff, which raises the possibility of intermingling between the two.

### The Footgun(s)

**<u>Under specific circumstances</u>** explored further below, backend environment variables could get baked into frontend JavaScript bundles. 

![footgun-viewsource](/assets/images/footgun-viewsource.png)

> This would expose sensitive internal information such as application secrets, build secrets, and additional credentials/access tokens to any application users with access to the frontend JS bundle.
{: .prompt-danger }

### Public Discourse

- 18/07/2024 - [https://github.com/vitejs/vite/issues/17710](https://github.com/vitejs/vite/issues/17710){:target="_blank"}  <sub>(semi-related)</sub>
- 19/03/2025 - [https://github.com/vitejs/vite/pull/19517](https://github.com/vitejs/vite/pull/19517){:target="_blank"}  <sub>(warning message added to v4.5.10)</sub>
- 09/02/2026 - [https://github.com/vitejs/vite/pull/21623/changes](https://github.com/vitejs/vite/pull/21623/changes){:target="_blank"}  <sub>(semi-related)</sub>

### Reproduction Steps

1. **Setup a throwaway node environment**
	1. make a new directory ie.`mkdir ~/vite-test && cd ~/vite-test`
	2. Download and run a Node docker container with current directory mounted  
	   `docker run -it --rm -v "$(pwd):/app" -w /app node:latest /bin/bash`
2. **Setup a Vite project in the directory** (in container)
	1. `npm create vite@5.2.0 -- -template vanilla-ts`
	2. Give the directory a random name of your choice, I went with `todo`
	3. I used the Typescript template for my testing but Javascript should work fine too
      ![footgun-vitecounter](/assets/images/footgun-vitecounter.png)
      _The template sets up a simple app with a counter that you can click on_
      <br/>


3. **Add `vite.config.ts`{: .filepath} file inside the newly templated directory** (on host)
  - Open a new shell and `cd ~/vitest/todo && touch vite.config.ts`:
   ```typescript
   import { defineConfig, loadEnv } from 'vite';  
   export default defineConfig(({ mode }) => {
     return {
       define: {
         "process.env": process.env
       }
     };
   });
   ```
   {: file='vite.config.ts'}

4. **Add a small `.env`{: .filepath} file with dummy data:** (on host)
   ```java
   VITE_FOO=barbarbar                             
   VITE_BAR=foofoofoo                           
   SECRET_KEY=iloveyou123                       
   VITE_HONEY=sugar                             
   API_TOKEN=apiapiapi     
   ```
   {: file='.env'}

   > You may need to adjust folder read/write permissions accordingly due to the directories installed by Docker process via mounting
   {: .prompt-info}

5. **Shell #2 into the Node docker container**
	1. Find the container's name under `docker container ls`
	2. `docker exec --it <CONTAINER_NAME> /bin/bash`
6. **Install packages** (in container)
	1. `cd /app/todo`
	2. Modify `package.json` to your target version of Vite, I used `5.1.8`
	3. `sed -i 's/"vite": "[^"]*"/"vite": "5.1.8"/' package.json`
	4. `npm install`
7. **Build the bundle** (in container)
	1. `npm run build -- -l info -w` (Build with logging enabled and watch for changes)
8. **Preview the bundle** (in container)
	1. `cd /app/todo`
	2. `npm run preview -- --host -d`
9. **Watch for changes in the bundle** (on host)
	1. Open a new shell and `cd ~/vite-test/todo`
	2. `watch -n 1 "cat dist/assets/*.js | tr ';' \"\\n\" | grep VITE ; echo && ls dist/assets/*.js"`
10. **Open `counter.ts`{: .filepath} on standby** (on host)
	1. Open a new shell and open `counter.ts`{: .filepath} for editing

    > If all goes well the setup should look like so:
    >![footgun-setup](/assets/images/footgun-setup.png)
    {: .prompt-tip}

11. **Add `console.log(import.meta.env)` to `counter.ts`{: .filepath}**

    ```typescript
    export function setupCounter(element: HTMLButtonElement) {
      let counter = 0
      const setCounter = (count: number) => {
        counter = count
        element.innerHTML = `count is ${counter}`
      }
      element.addEventListener('click', () => setCounter(counter + 1))
      console.log(import.meta.env) // <------
      setCounter(0)
    }
    ```
    {: file='counter.ts'}

12. **Observe rebuild of bundle on file save**

    ```go
    build started...

    ✓ 1 modules transformed.
    dist/index.html                 0.46 kB │ gzip: 0.29 kB
    dist/assets/index-Cz4zGhbH.css  1.21 kB │ gzip: 0.62 kB
    dist/assets/index-Bd-pKGJy.js   3.05 kB │ gzip: 1.64 kB
    built in 32ms.
    ```
    {: file='build watch output'}  

13.  **Observe `VITE_*` and default environment variables pop up in the watch window**  

     ```shell
     Every 1.0s: cat dist/assets/*.js | tr ';' "\n" | grep VITE ; echo && ls dist/assets/*.js                      kali: Sat Mar 18 10:02:22 2026
        
     var a={VITE_FOO:"barbarbar",VITE_BAR:"foofoofoo",VITE_HONEY:"sugar",BASE_URL:"/",MODE:"production",DEV:!1,PROD:!0,SSR:!1}
        
     dist/assets/index-Bd-pKGJy.js
     ```
     {: file='bundle watch output'}  

14.   **Move `console.log(import.meta.env)` to a new function within `counter.ts`{: .filepath}**  

      ```typescript
      export function setupCounter(element: HTMLButtonElement) {  
        let counter = 0
        const setCounter = (count: number) => {
          counter = count
          element.innerHTML = `count is ${counter}`
        }

        element.addEventListener('click', () => setCounter(counter + 1))
        console.log(import.meta.env)
        setCounter(0)
      }

      function foo() {                  // <-----
        console.log(import.meta.env)    // <-----
      }                                 // <-----
      ```
      {: file='counter.ts'}


15.  **Observe rebuild of bundle on file save**
16.  **Observe `VITE_*` and default environment variables gone from the watch window**

> This happens because the `foo` function is not `exported` and is therefore not expected to be used in the frontend. The `setupCounter` function is imported on line 4 of `main.ts`{: .filepath}, which in turn is called by `index.html`{: .filepath}. Thus, any function that is interacted with on the frontend will be bundled.
{: .prompt-info}

#### The Missing Piece ( A Hypothesis)

> I was stuck for awhile wondering how despite `process.env` being included into Vite; it wasn't automatically making its way into the frontend. Since abusing `import.meta.env` did not work, I went back to playing around with `process.env`.
{: .prompt-warning}


1. **Observe `process.env` in vite preview window:**
	![footgun-process.png](/assets/images/footgun-process.png)
14. **Add `console.log(process.env)` to `counter.ts`{: .filepath}**
    ```typescript
    export function setupCounter(element: HTMLButtonElement) {  
      let counter = 0
      const setCounter = (count: number) => {
        counter = count
        element.innerHTML = `count is ${counter}`
      }
      
      element.addEventListener('click', () => setCounter(counter + 1))
      console.log(import.meta.env)
      console.log(process.env)      // <-----
      setCounter(0)
    }
    ```
    {: file='counter.ts'}

15. **Observe rebuild of bundle on file change**
16. **Observe backend environment variables pop up in the watch window**
	![footgun-boom](/assets/images/footgun-boom.png)
  _ლ(ಠ_ಠ ლ)_

> Note that if you exit the build watch and try to do fresh build of the bundle, it will fail with:  
> ```
> src/counter.ts:8:15 - error TS2580: Cannot find name 'process'.   
> Do you need to install type definitions for node?   
> Try `npm i --save-dev @types/node`.  
>  
> 8   console.log(process.env) 
> ```  
> However, *for whatever reason* - the build process works if you :
> - build first with no errors
> - have watch enabled for the build
> - add in `console.log(process.env)` + `console.log(import.meta.env)`
> - save and trigger a re-build via watch 
>
> > This was the only way I've seen so far to reproduce after much trial and error. There might be other ways too but will be left as an exercise for the reader.
{: .prompt-warning}

#### Does this method work on latest Vite?

1. **Navigate back to container pwd** (in container)
	1. cd `/app`
2. **Install new Vite project using newest template** (in container)
	1. (Newest Vite Template at time of writing is 9.0.3)
	2. `npm create vite@9.0.3 -- -template vanilla-ts`
3. **Observe Vite version - `Vite v8.0.1`**
4. **Continue with remaining steps as usual**
5. **Observe warning message on build**
6. **Observe outcome in JS bundle**

- [x] **The answer is yes for up to v8.0.1**


### How bad is it?

Currently, a sample GitHub [search](https://github.com/search?q=path%3A%2Fvite%5C.config%5C.%5Btj%5Ds%2F+%2Fimport+%5C%7B+%5B%5E%7B%5D*defineConfig%5B%5E%7D%5D*%5C%7D+from+%5B%27%22%5Dvite%5B%27%22%5D%2F+AND+%2F%5B%22%27%5Dprocess.env%5B%27%22%5D%3A.*process.env%2F&type=code){:target="_blank"} reveals ~5.2k public repos using this configuration:

![footgun-capacitor](/assets/images/footgun-vitejsprocessenv.png)
> However, this doesn't account for whether the repos include `console.log(process.env)`. I grabbed a sample size of 120+ repos from GitHub search and grepped for any such instances but found none.

Newer versions of Vite will also warn you when the unsafe config is present, as per [pull](#public-discourse-1) above.
![footgun-vitewarning](/assets/images/footgun-vitewarning.png)

---

## 3. ViteJS loadEnv Empty Prefix

### What is loadEnv?

This is a function that allows developers to load in environment variables based on different environment files such as `.env`, `.env.production`, `.env.staging`, etc. It is great for having the stack run properly no matter which environment it is dropped into.

- According to official [docs](https://vite.dev/guide/api-javascript#loadenv){:target="_blank"}:
: Load .env files within the envDir. By default, only env variables prefixed with `VITE_` are loaded, unless prefixes is changed.

- Function definition:
  ```typescript
  function loadEnv(
    mode: string,
    envDir: string,
    prefixes: string | string[] = 'VITE_',
  ): Record<string, string>
  ```
  {: file='loadEnv definition'}

As documented above, Vite tries to keep developers safe by only including environment variables prefixed with `VITE_` by default. This makes it a more explicit action that can't be misunderstood.

However, developers apparently still needed to inject runtime environment variables which depending on how much the environment changes, may not already have the `VITE_` prefix. Leaving the prefix empty is the "easiest" way to solve this problem, but it is not the only way.

> In newer versions of Vite, Vite will refuse to build if you specify an empty prefix via the `envPrefix`([docs](https://vite.dev/config/shared-options#envprefix){:target="_blank"}) option, since an empty prefix would match all environment variables.
> ![footgun-prefix](/assets/images/footgun-prefix.png)   
{: .prompt-tip}

Currently, no such error is given when an empty prefix is passed to `loadEnv`'s third parameter.

### The Footgun

> The risk and impact here are very similar to the `process.env` observation above.

As discovered through several cases below, developers that:
- copied code samples from the official documentation 
- blindly used LLM-generated content
- accepted the risk for the easiest solution  

were likely to end up having the code below:
```typescript
import { defineConfig, loadEnv } from 'vite';                       
                                                                    
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '')

  return {
    define: {
        "process.env": env
    }
  };
});
```
  {: file='BAD vite.config.ts'}
- _This code directly passes `env` to `process.env`, which introduces unnecessary risk._


This is different than the official code [sample](https://vite.dev/config/#using-environment-variables-in-config){:target="_blank"}:
```typescript
import { defineConfig, loadEnv } from 'vite'

export default defineConfig(({ mode }) => {
  // Load env file based on `mode` in the current working directory.
  // Set the third parameter to '' to load all env regardless of the
  // `VITE_` prefix.
  const env = loadEnv(mode, process.cwd(), '')
  return {
    define: {
      // Provide an explicit app-level constant derived from an env var.
      __APP_ENV__: JSON.stringify(env.APP_ENV),
    },
    // Example: use an env var to set the dev server port conditionally.
    server: {
      port: env.APP_PORT ? Number(env.APP_PORT) : 5173,
    },
  }
})
```
  {: file='GOOD vite.config.ts'}
- _Although environment variables are loaded into the `env` variable, it is never directly passed to `process.env`_



### Public Discourse

- 15/05/2024 - [https://github.com/vitejs/vite/issues/16686](https://github.com/vitejs/vite/issues/16686){:target="_blank"}
- 24/10/2024 - [https://github.com/vitejs/vite/pull/18441](https://github.com/vitejs/vite/pull/18441){:target="_blank"}
- 25/02/2025 - [https://github.com/vitejs/vite/pull/19510](https://github.com/vitejs/vite/pull/19510){:target="_blank"}
- 25/08/2025 - [https://github.com/vitejs/vite/pull/20624/changes](https://github.com/vitejs/vite/pull/20624/change){:target="_blank"} <sub>(addressed vague documentation)</sub>

## Reproduction steps

The steps for testing and reproduction are basically the same as for above:
- just swap out the code for `loadEnv` in `vite.config.ts` re. the bad code sample above
- the same `console.log(process.env)` steps also apply

### How bad is it?

Currently, a sample GitHub [search](https://github.com/search?q=path%3A%2Fvite%5C.config%5C.%5Btj%5Ds%2F+%2Fimport+%5C%7B+%5B%5E%7B%5D*defineConfig%5B%5E%7D%5D*%5C%7D+from+%5B%27%22%5Dvite%5B%27%22%5D%2F+AND+%2FloadEnv%5C%28.*%2C%5B%27%22%5D%5B%27%22%5D%5C%29%2F+AND+%2Fprocess.env%5B%5E%5C.%5D%2F&type=code){:target="_blank"} reveals ~81 public repos using this configuration:  
![footgun-loadenv](/assets/images/footgun-vitejsloadenv.png)

<div class="divider"></div>

## Conclusion

It's not much but I hope this blog can help bring more awareness to the unexpected and sometimes dangerous side-effects of footguns. It seems that modern web technology is continuing to trend toward the direction of blurring the lines between the frontend and backend, so these types of footguns may continue to surface in the coming future.

I have written up custom semgrep [rules](https://github.com/kawing-ho/semgrep-rules){:target="_blank"} which you can use to check if your local repositories are affected.
