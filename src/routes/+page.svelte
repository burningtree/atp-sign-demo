<script>
    import { onMount } from "svelte";
    import { verifySignature } from "@atproto/crypto";
    import { BskyAgent } from "@atproto/api";
    import { fromString } from "uint8arrays";

    import { BrowserOAuthClient } from "@atproto/oauth-client-browser";
    import ClientMetadata from "$lib/client-metadata.json";

    let agent;
    let oauthClient;
    let session = $state();

    let pdsService = $state("bsky.social");
    let username = $state("");
    let password = $state("");

    let signDid = $state("");
    let signMessage = $state(
        "Signing a message is proof that it’s truly from you, like leaving your mark on a sealed letter.",
    );
    let signSig = $state("");

    let verifyDid = $state("did:plc:524tuhdhh3m7li5gycdn6boe");
    let verifyMessage = $state(
        "Signing a message is proof that it’s truly from you, like leaving your mark on a sealed letter.",
    );
    let verifySig = $state(
        "d0f8b3cf6d89ca61ae493d3e1a0f5c24d1db29bf181e1c530099678f0418edb31223a2221999c57a1f1efdfd86b26b3370206691ca7aa68a7abb09761eb6a53e",
    );
    let verifyValid = $state(null);
    let verifyProcessing = $state(false);
    let verifyError = $state(false);

    async function login() {
        agent = new BskyAgent({
            service: "https://" + pdsService,
        });
        const auth = await agent.login({ identifier: username, password });
        localStorage.setItem("session", JSON.stringify(auth.data));
        afterLogin(auth.data);
    }

    async function logout() {
        localStorage.removeItem("session");
        afterLogout();
    }

    async function loginOauth() {
        try {
            await oauthClient.signIn(signDid, {
                state: "some value needed later",
                prompt: "none",
                ui_locales: "en",
                signal: new AbortController().signal,
            });
        } catch (err) {
            console.log(err);
            console.log(
                'The user aborted the authorization process by navigating "back"',
            );
        }
    }

    function signParams() {
        return {
            did: signDid,
            message: signMessage,
        };
    }

    function signReset() {}

    async function sign() {
        signReset();

        let resp;
        try {
            resp = await fetch(
                "https://pds.gwei.cz/xrpc/com.atproto.identity.signMessage",
                {
                    method: "post",
                    headers: {
                        "content-type": "application/json",
                        authorization: `Bearer ${session.accessJwt}`,
                    },
                    body: JSON.stringify(signParams()),
                },
            );
        } catch (e) {}
        const json = await resp.json();
        signSig = json.sig;
    }

    function verifyReset() {
        verifyValid = null;
        verifyProcessing = false;
        verifyError = false;
    }
    function verifyParams() {
        return {
            did: verifyDid,
            message: verifyMessage,
            sig: verifySig,
        };
    }

    async function verify() {
        verifyReset();
        const { did, message, sig } = verifyParams();

        // load DID document with public keys
        let resp;
        try {
            resp = await fetch(`https://plc.directory/${did}/data`);
        } catch (e) {}
        const didDoc = await resp.json();

        const messageBytes = fromString(didDoc.did + "\n" + message, "utf-8");
        const sigBytes = fromString(sig, "hex");
        console.log(messageBytes);

        const validator = async () => {
            for (const key of didDoc.rotationKeys) {
                if (await verifySignature(key, messageBytes, sigBytes)) {
                    return true;
                }
            }
            return false;
        };

        verifyValid = await validator();

        return null;
    }

    async function verifyOnPDS() {
        verifyReset();

        let resp;
        try {
            resp = await fetch(
                "https://pds.gwei.cz/xrpc/com.atproto.identity.verifyMessage?" +
                    new URLSearchParams(verifyParams()).toString(),
            );
        } catch (e) {
            verifyError = "xx";
        }
        const { valid } = await resp.json();
        verifyValid = valid;
        verifyProcessing = false;
    }

    async function signAndVerify() {
        await sign();
        verifyDid = signDid;
        verifyMessage = signMessage;
        verifySig = signSig;
        await verifyOnPDS();
    }

    function afterLogin(currentSession) {
        session = currentSession;
        signDid = session.did;
    }

    function afterLogout() {
        session = null;
        signDid = null;
    }

    onMount(async () => {
        verify();

        agent = new BskyAgent({
            service: "https://" + pdsService,
        });

        const prevSession = JSON.parse(localStorage.getItem("session"));
        if (prevSession) {
            agent.resumeSession(prevSession);
            afterLogin(prevSession);
        }

        /*oauthClient = new BrowserOAuthClient({
            handleResolver: 'https://api.bsky.app',
            // Only works if the current origin is a loopback address:
            clientMetadata: undefined,
        })*/
    });
</script>

<div class="p-4">
    <h1 class="text-3xl">AT Protocol Signatures</h1>
    <div class="grid grid-cols-1 lg:grid-cols-2 gap-4 mt-6">
        <div class="">
            <h2 class="text-2xl mb-2">Sign Message</h2>
            <div class="grid grid-cols-1 gap-4">
                <div>
                    {#if session}
                        <div class="font-bold">DID:</div>
                        <div>
                            <input
                                type="text"
                                class="border p-2 w-full"
                                disabled
                                bind:value={signDid}
                            />
                        </div>
                        <button
                            class="p-2 rounded bg-blue-600 text-white active:bg-blue-300 mt-2"
                            onclick={logout}>Log out</button
                        >
                    {:else}
                        <div>
                            <div>
                                <div>PDS</div>
                                <div>
                                    <input
                                        type="text"
                                        class="border p-2 full"
                                        name="login"
                                        bind:value={pdsService}
                                    />
                                </div>
                            </div>
                            <div>
                                <div>Handle</div>
                                <div>
                                    <input
                                        type="text"
                                        class="border p-2 full"
                                        name="login"
                                        bind:value={username}
                                    />
                                </div>
                            </div>
                            <div>
                                <div>App Password</div>
                                <div>
                                    <input
                                        type="password"
                                        class="border p-2 full"
                                        name="password"
                                        bind:value={password}
                                    />
                                </div>
                            </div>
                        </div>
                        <button
                            class="p-2 rounded bg-blue-600 text-white active:bg-blue-300 mt-2"
                            onclick={login}>Sign In</button
                        >
                    {/if}
                </div>
                <div>
                    <div class="font-bold">Message:</div>
                    <textarea
                        class="border w-full p-2"
                        rows="5"
                        bind:value={signMessage}
                    />
                </div>
                <div>
                    <button
                        type="submit"
                        class="p-2 rounded bg-blue-600 text-white active:bg-blue-300 disabled:bg-gray-200"
                        disabled={!session}
                        onclick={sign}>Sign</button
                    >
                    <button
                        type="submit"
                        class="p-2 rounded bg-blue-600 text-white active:bg-blue-300 disabled:bg-gray-200"
                        disabled={!session}
                        onclick={signAndVerify}>Sign & Verify</button
                    >
                </div>
                <div>
                    <div class="font-bold">Signature:</div>
                    <textarea
                        class="border w-full p-2 font-mono"
                        rows="3"
                        bind:value={signSig}
                    ></textarea>
                </div>
            </div>
        </div>
        <div>
            <h2 class="text-2xl mb-2">Verify Message</h2>
            <div class="grid grid-cols-1 gap-4">
                <div>
                    <div class="font-bold">DID:</div>
                    <div>
                        <input
                            type="text"
                            class="border p-2 w-full"
                            bind:value={verifyDid}
                        />
                    </div>
                </div>

                <div>
                    <div class="font-bold">Message:</div>
                    <textarea
                        class="border w-full p-2"
                        rows="5"
                        bind:value={verifyMessage}
                    />
                </div>
                <div>
                    <div class="font-bold">Signature:</div>
                    <textarea
                        class="border w-full p-2 font-mono"
                        rows="3"
                        bind:value={verifySig}
                    />
                </div>
                <div>
                    <button
                        type="submit"
                        class="p-2 rounded bg-blue-600 text-white active:bg-blue-300 disabled:bg-gray-200"
                        onclick={verify}
                        disabled={verifyProcessing ||
                            !verifyMessage ||
                            !verifySig ||
                            !verifyDid}>Verify</button
                    >
                    <button
                        type="submit"
                        class="p-2 rounded bg-blue-600 text-white active:bg-blue-300 disabled:bg-gray-200"
                        onclick={verifyOnPDS}
                        disabled={verifyProcessing ||
                            !verifyMessage ||
                            !verifySig ||
                            !verifyDid}>Verify on PDS</button
                    >
                </div>
                {#if verifyValid !== null}
                    <div class="mt-2 text-lg">
                        {#if verifyValid === true}
                            <div class="text-green-600">
                                ✓ Signature is valid!<br />DID:
                                <a
                                    href="https://internect.info/did/{verifyDid}"
                                    class="underline hover:no-underline"
                                    target="_blank">{verifyDid}</a
                                >
                            </div>
                        {/if}
                        {#if verifyValid === false}
                            <div class="text-red-600">
                                ❌ Signature is invalid!
                            </div>
                        {/if}
                    </div>
                {/if}
                {#if verifyError}
                    <div class="mt-2">
                        Error: {verifyError}
                    </div>
                {/if}
            </div>
        </div>
    </div>
</div>
