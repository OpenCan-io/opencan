![Screenshot](img/dfinity-logo.svg)

## Getting Started

## Requirements

Install the beta version of the DFINITY Canister SDK

<div class="bd-example"><pre><code>
DFX_VERSION=0.7.0-beta.8 sh -ci "$(curl -fsSL https://sdk.dfinity.org/install.sh)" 
</pre></code></div>


## Get Started

With NPM:

<div class="bd-example"><pre><code>
npx create-ic-app
</pre></code></div>


![Screenshot](img/setup.png)

After the setup is done
<div class="bd-example"><pre><code>
cd test-project
npm install
</pre></code></div>
Start the backend
<div class="bd-example"><pre><code>
dfx start --background
dfx canister create --all
dfx build
dfx canister install --all
</pre></code></div>
Start the frontend
<div class="bd-example"><pre><code>
npm run dev
</pre></code></div>
Thats it! The counter demo is persisted on your local Internet Computer.

![Screenshot](img/example.png)

## [Experimental] Watch mode for canisters
This will auto build and install your canisters whenever you make a change to them
<div class="bd-example"><pre><code>
npm run watch
</pre></code></div>

![Screenshot](img/create-ic-app.gif)

## Adding / Removing / Renaming Canisters

You will have to edit 3 files:

* dfx.json
* tsconfig.json
* src/agent.ts

More detailed instructions later. For now see how it's done in those files for the example counter canister.

## Deploying Asset Canister

Run the following commands
<div class="bd-example"><pre><code>
dfx build
dfx canister install --all --mode=reinstall
</pre></code></div>
Then grab the asset canister ID from the output. Shown here:
![Screenshot](img/deployCanister.png)

Now you can visit the following url in your browser to see it running on your local internet computer:

<code>http://localhost:8000?canisterId=<CopiedCanisterId\></code>

## Community templates / Contributions
Want to contribute your own template? Find something that could be improved? Repo is open for PRs! Happy to assist you in this. You'll receive full credit for your contribution of course.

### Contributions
@ferMartz (React JS + Tailwind template)