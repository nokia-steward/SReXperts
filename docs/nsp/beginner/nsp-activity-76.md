# LSP Visualization

|     |     |
| --- | --- |
| **Activity name** | LSP Visualization |
| **Activity ID** | 74 |
| **Short Description** | Explore and visualize LSPs, create Path Profiles to control LSP optimization and provision a PCC-initiated LSP |
| **Difficulty** | Beginner |
| **Topology Nodes** | :material-router: PE1, :material-router: PE2, :material-router: P1, and :material-router: P2 |
| **References** | [Developer Portal](https://network.developer.nokia.com) |

## Objective

Managing SR-TE and MPLS tunnels across a multi-router network is one of those tasks that looks simple until you are doing it at scale. How many LSPs are active right now? Which path does each one take? Is that path still optimal after a link went down last night? In a traditional CLI workflow, answering any of those questions means opening device sessions one by one and piecing together the picture yourself.

This activity shows you a better way. You will use NSP as a single pane of glass to discover and visualize LSPs across the topology, define Path Profiles to control how the PCE optimizes your tunnels, and provision a new PCC-initiated LSP, all without touching a single CLI session. By the end, you will have a clear picture of how centralized visibility and policy-driven path control can reduce the time and risk involved in day-to-day LSP operations.

## Technology explanation

### PCE, PCC, and PCEP

In a PCE-based network, path computation is split between two roles:

- **PCE (Path Computation Element)** — a centralized controller (NSP in this case) that computes optimal paths based on topology, constraints, and optimization objectives. The PCE has a global view of the network that no single router possesses.

- **PCC (Path Computation Client)** — a router (such as PE1) that delegates path computation to the PCE. The PCC signals and maintains the LSP, but relies on the PCE to decide *where* it goes.

**PCEP (Path Computation Element Communication Protocol)** is the protocol that carries messages between PCC and PCE. Through PCEP, a PCC can request a path, receive a computed route, and hand over ongoing control of that LSP to the PCE — a mode called *delegation*. This delegation is what makes dynamic re-optimization possible: when topology changes (a link fails, latency spikes), the PCE can push a new path to the PCC without any manual intervention.

### Path Profiles

Path Profiles in NSP Path Control allow you to define how the PCE (Path Computation Element) or PCC (Path Computation Client) optimizes LSPs — controlling objectives, constraints, bandwidth strategies, and protection. Once created, a Path Profile can be referenced by any LSP to enforce consistent computation behavior.

### Model-Driven Configurator (MDC)

MDC allows you to directly push YANG-modeled configuration to a device through the NSP WebUI without writing any code.

## Tasks

/// warning
This activity requires NSP with **VSR NRC** enabled. Use one of the following instances:

|     |     |     |
| --- | --- | --- |
| **Server** | `nsp1.srexperts.net` | `nsp2.srexperts.net` |
| **Instance** | 2 | 42 |
| **Username** | `pathcontrol` |
| **Password** | `EVENT_PASSWORD` |

Remember that you are using a shared NSP system. Include your group number in every activity you perform.
///

**You should read these tasks from top to bottom before beginning the activity.**

It is tempting to skip ahead, but tasks may require you to have completed previous tasks before tackling them.

### Quick start on NSP Web UI

|     |     |
| --- | --- |
| **NE Session** | `☰` → `Network Search and Inventory` → find your group’s PE node (for example `g7-pe1`) → open the row context menu `⋮` → `Open in NE Session`. |
| **NSP Help** | `?` icon at the top right for context-aware quick help and to open the Help Center. On some pages, `?` also links directly to related Help Center articles. |
| **Path Control** | `☰` → `Path Control` |
| **Model-Driven Configurator** | `☰` → `Model Driven Configurator` |

/// note
This activity is intentionally open-ended. There is no single correct answer or final deliverable. Focus on exploring, asking questions, and documenting what you find. Your observations are the outcome.
///

### Locate and Inspect Active LSPs

Now let's find the LSPs running in the network.

1. Navigate to **Path Control**.
   - What information can you extract from this menu?
2. Navigate to the **LSPs** menu from the dropdown.
3. Review the list of LSPs currently visible:
    - How many LSPs are active?
    - What are their source and destination nodes?
    - What is the operational state of each LSP?
4. Select one LSP and open its detail view. Take note of:
   - The **actual hop-by-hop path** through the network
   - The **bandwidth reservation** (if available)
   - The **operational state** (up, down, degraded)
5. Try to identify whether the LSP is using a **primary** or **secondary** path.

/// note
Not all LSPs may have explicit paths configured. Dynamic LSPs rely on RSVP-TE or SR-TE computation. Note any differences you observe between LSP types.
///

### Query LSP Data via REST API

NSP exposes LSP operational data through its Path Control REST API. Use **Postman** or the **NSP API documentation** at `https://{{server}}/sdn/doc` to explore and run requests interactively — the response is easier to navigate than raw `curl` output.

/// note
The API documentation page (`/sdn/doc`) is the NSP **API Explorer**: browse the Path Control endpoints, inspect request/response schemas, and try calls from your browser.
///

#### Step 1 — Authenticate

In Postman (or `curl`), obtain a bearer token using the credentials from the warning box above:

/// note | Example: Authenticate and get a token

=== "Linux / macOS"
    ```bash
    curl -k -X POST https://{{server}}/rest-gateway/rest/api/v1/auth/token \
      -H "Content-Type: application/json" \
      --user "pathcontrol:$EVENT_PASSWORD" \
      -d '{"grant_type": "client_credentials"}'
    ```

=== "Windows (PowerShell)"
    ```powershell
    curl.exe -k -X POST https://{{server}}/rest-gateway/rest/api/v1/auth/token `
      -H "Content-Type: application/json" `
      --user "pathcontrol:$EVENT_PASSWORD" `
      -d '{\"grant_type\": \"client_credentials\"}'
    ```

Save the returned `access_token` as you will need it for all subsequent requests.
///

#### Step 2 — Query SR-TE LSP paths

/// details | Retrieve SR-TE LSP paths (hint)
   type:hint

=== "Linux / macOS"
    ```bash
    curl -k -X GET "https://{{server}}/sdn/api/v4/mpls/lsp-paths" \
      -H "Authorization: Bearer YOUR_TOKEN" \
      -H "Accept: application/json"
    ```

=== "Windows (PowerShell)"
    ```powershell
    curl.exe -k -X GET "https://{{server}}/sdn/api/v4/mpls/lsp-paths" `
      -H "Authorization: Bearer YOUR_TOKEN" `
      -H "Accept: application/json"
    ```
///

Explore the response and look for:

- LSP name and identifier
- Ingress and egress node
- Path hops
- Operational state fields

### Create a Path Profile in NSP

Navigate to **Path Control** → **Path Profiles** (dropdown), then click the **+** button in the top right (tooltip: **Create Policy**).

Configure the Path Profile using the parameters below. Not all fields are mandatory. Focus on the highlighted ones for this activity.

| Parameter | Description | Value for this Activity |
|-----------|-------------|------------------------|
| **Reserved Profile ID** | Whether this profile assumes the role of the default path profile policy | Leave unchecked |
| **Name** | Name of the path profile policy | `hackathon-path-profile-groupXX` |
| **Profile ID** | Identifier used to associate this profile with LSP path computation | Choose a unique integer (e.g., `100`) |
| **Description** | Optional free-text description of the path profile | Leave empty |
| **Bi-directional** | Bidirectional mode for path computation, if any | `No` |
| **Disjoint** | Disjoint mode for path computation, if any | `No` |
| **Optimize On (Objective)** | Primary goal when identifying paths | `TE Metric` |
| **Bandwidth Strategy** | Strategy for bandwidth collection | `Standard` |
| **Keep Bandwidth Reservation on Failure** | Whether to keep bandwidth reserved when the LSP fails | Leave unchecked |
| **Explicit Route Strategy** | Explicit route strategy for the service | `Standard` |
| **Control Route Strategy** | Strategy used when rerouting a path | `Standard` |
| **SID Protection Strategy** | Extent of SID protection when routing | `Standard (Protected Preferred)` |
| **Max Hops (Span)** | Maximum number of hops (nodes) to consider | `10` |
| **Max Cost** | Maximum IGP link metric sum to consider | Leave default |
| **Max TE Metric** | Maximum TE metric sum to consider | Leave default |
| **Max Latency (microseconds)** | Maximum latency to consider | Leave default |
| **Latency Threshold (microseconds)** | When to re-signal an LSP optimized on latency | Leave default |
| **Exclude Route Objects** | Nodes to exclude from the path, if possible | Leave empty |
| **Include Route Objects** | Nodes to include in the path, if possible | Leave empty |

/// tip
Replace `XX` in the profile name with your group number to avoid conflicts with other teams sharing the same NSP instance.
///

/// note
The **Profile ID** is the key field that links this policy to an LSP. Make a note of the value you chose—you will need it when you assign the path profile while provisioning your PCC-initiated LSP below.
///

Once you have filled in the parameters, click **Save** and verify that the new Path Profile appears in the list.

### Provision a PCC-Initiated LSP

Now that you have a Path Profile, create a **PCC-initiated SR-TE LSP** on PE1 and attach it to the profile you just created. Use **MDC** (Model Driven Configurator) as shown below.

1. Navigate to **Model Driven Configurator** and select **PE1** as the target device.
2. Navigate to the configuration tree: `configure` > `router` > `Base` > `mpls` > `lsp`.
3. Click **Create LSP** and fill in the following fields:

| Parameter | Value |
|-----------|-------|
| `lsp-name` | `hackathon-lsp-groupXX` |
| `type` | `p2p-sr-te` |
| `admin-state` | `enable` |
| `from` | PE1 system IP |
| `to` | PE2 system IP |
| `pce-control` | `true` |
| `pce-report` | `true` |
| `path-computation-method` | `pce` |
| `fallback-path-computation-method` | `local-cspf` |

4. Under the LSP, add a **primary path** with the following settings:

| Parameter | Value |
|-----------|-------|
| `path-name` | `loose` |
| `admin-state` | `enable` |

5. Also add a **path-profile** entry:

| Parameter | Value |
|-----------|-------|
| `profile-id` | The Profile ID you set when creating the Path Profile above |
| `path-group` | Leave default or ask your facilitator |

6. Click **Deploy** and verify the configuration was applied successfully on PE1.

/// tip
After deploying, navigate back to **Network** > **MPLS** and confirm your new LSP appears in the list with an **Up** operational state.
///

### Play, Reflect, and Share Observations

This final task combines a hands-on exploration exercise with a team discussion and note-taking activity. There are no right or wrong answers — your insights and observations are the goal.

#### Part 1 — Free exploration: try to break things

Now that you have a solid understanding of LSPs and Path Profiles, it is time to experiment freely. Push the network to its limits and observe how NSP responds.

Here are some ideas to get you started:

- Change the optimization objective of an existing LSP and observe whether it gets rerouted
- Shut down a port along an active LSP path and see how the network reacts
- Modify a Path Profile shared by multiple LSPs and observe the impact
- Create conflicting LSPs with competing optimization objectives and see what happens
- Remove a link along an active LSP path and observe the ripple effect

/// note
Take notes on what you try and what you observe — these will be valuable for the discussion in Part 2.
///

#### Part 2 — Team Discussion

As a team, discuss and document your answers to the following.

**LSP visibility**

- Was the LSP information in NSP complete and accurate compared to what you expected?
- How easy was it to trace an LSP end-to-end through the topology?
- Did your newly created LSP appear correctly on the topology map?

**Path Profiles**

- How does using a Path Profile change the way you think about LSP provisioning at scale?
- What would happen if two LSPs with conflicting optimization objectives shared the same Path Profile?

## Summary

Congratulations! In this activity you explored how NSP surfaces, controls, and provisions LSP data across a network topology. Key takeaways include:

- **Centralized visibility matters** — NSP eliminates the need to SSH into individual nodes to understand LSP state, dramatically reducing the time to answer basic operational questions.
- **Path Profiles unlock consistency** — By decoupling optimization policy from individual LSP configuration, Path Profiles allow network teams to enforce consistent computation behavior across hundreds of LSPs without touching each one individually.
- **PCE delegation is powerful** — Handing path computation to the PCE with `pce-control: true` means the network can react dynamically to topology changes, without requiring manual re-provisioning.
- **Topology context is key** — Seeing an LSP highlighted on a topology map immediately reveals dependencies and potential failure impact in a way that raw CLI output cannot.
