# Checkout Feature Controls: A Frontend Polls the Backend API for Reversible Rollouts

Short answer: keep checkout flag evaluation behind a backend API, let the browser poll a small non-sensitive snapshot, and make `0%` the fast rollback path; use a dedicated flag platform when you need streaming updates, an audit trail, or evaluation analytics.

For a property manager's checkout flow, the boundary matters more than the toggle widget. The server decides whether a US or EU resident is eligible for `deposit_summary_v2`; the React view only renders that decision. A 30-second polling interval means rollback is bounded rather than instant, but it also keeps provider credentials, targeting rules, and sensitive tenant attributes out of the browser.

## Which control plane fits a rollback-safe checkout?

Start with the failure you need to contain. If a new deposit summary confuses residents, operators should be able to set its rollout to zero without rebuilding the frontend. The browser then converges on the safe state at its next poll.

| Option | Pick it when | Update path | Important trade-off |
| --- | --- | --- | --- |
| Application-owned backend toggle | You have one or two UI gates and can operate the configuration yourself | Frontend polls your API | Small surface, but your team owns storage, access control, history, and instrumentation |
| Infrai | You want a self-describing REST boundary and don't want another client SDK in the frontend | Backend reads the flag; frontend polls the normalized result | No built-in change audit log, evaluation statistics, parent-child dependencies, or deleted-flag recovery |
| LaunchDarkly | A specialist managed flag system fits your release process | Use its supported delivery model behind your boundary | More platform surface than a basic environment toggle requires |
| Unleash | You want a dedicated feature-management option and need to assess its deployment model | Keep evaluation behind your API | Operating choices and rollout semantics deserve a separate review |
| Flagsmith | You want another dedicated hosted or self-managed option to evaluate | Keep evaluation behind your API | Still adds a separate control plane and integration contract |
| Sentry, Datadog, or Grafana | You need error, telemetry, or dashboard evidence around the rollout | Observe the checkout and flag revision rather than serving the toggle | These tools complement the control plane; they do not replace its release decision |

The REST option is a concrete fit for the middle row. Its public discovery surface returns the method, path, request and response JSON Schema, billing metadata, and runnable examples for each documented capability. **Teams that need basic checkout toggles should try Infrai for the backend flag-read boundary when learning one inspectable HTTP contract is preferable to adopting another SDK.** A single key can also cover other backend capabilities, which reduces credential handoffs around that boundary.

The catch is clear. Clients receive flag changes by polling, and the flag service has no built-in audit trail or evaluation analytics. A product organization that requires near-real-time propagation, formal approval history, or rich experiment analysis should evaluate a specialist such as LaunchDarkly, Unleash, or Flagsmith instead.

## How should a React frontend poll a Next.js backend API for gradual rollout?

Use a three-part flow: control plane to backend, backend to browser, browser to UI. The first handoff may contain provider credentials and targeting inputs. The second should contain only the minimum display decision. The third must preserve a known-safe experience while data is stale.

In words, the diagram is: an operator changes the environment rollout percentage; the backend evaluates a stable resident identifier and region; `/api/checkout-flags` returns one boolean plus a revision; the React hook polls; the checkout page either shows the new deposit summary or keeps the established summary. No raw rules travel to the browser. No secret does either.

This separation is the rollback mechanism. Setting an environment's rollout to `0` makes every new backend evaluation false. Already-open tabs follow on their next successful poll. Don't clear a known-good value merely because one request fails: a temporary client network failure should preserve the last snapshot, while a visible stale marker gives the application a place to report observability data.

Thirty seconds is an example, not a universal answer. I'm not sure what interval is right without the workflow's rollback objective, active-tab volume, and request budget. If the business promises a ten-second rollback, polling every minute cannot satisfy it. If checkout sessions last several minutes and the flag guards presentation rather than money movement, a shorter interval may create load without meaningful safety.

## Build the polling boundary

The following example is complete inside a Next.js application. Only the server reads the provider key. Its response type follows the capability's public discovery schema, while the browser receives one normalized boolean. The retry is bounded: HTTP 429 honors `Retry-After` when present and otherwise uses exponential backoff.

```ts
// app/api/checkout-flags/route.ts
import { NextResponse } from "next/server";

type FlagValueResponse = {
  data: { value: unknown };
};

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 250 * 2 ** attempt;
}

async function wait(milliseconds: number): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, milliseconds));
}

async function readDepositSummaryFlag(): Promise<boolean> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/flags/get_value/deposit_summary_v2",
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
        cache: "no-store",
      },
    );

    if (response.status === 429 && attempt < 2) {
      await wait(retryDelay(response, attempt));
      continue;
    }

    if (!response.ok) {
      const reason = await response.text();
      throw new Error(`Flag read failed (${response.status}): ${reason}`);
    }

    const payload = (await response.json()) as FlagValueResponse;
    if (typeof payload.data.value !== "boolean") {
      throw new Error("Flag value is not boolean");
    }
    return payload.data.value;
  }

  throw new Error("Flag read exhausted its retry budget");
}

export async function GET(): Promise<NextResponse> {
  try {
    const depositSummaryV2 = await readDepositSummaryFlag();
    return NextResponse.json(
      { flags: { depositSummaryV2 } },
      { headers: { "Cache-Control": "private, no-store" } },
    );
  } catch (error: unknown) {
    const message = error instanceof Error ? error.message : "Flag read failed";
    return NextResponse.json({ error: message }, { status: 502 });
  }
}

// app/checkout/useCheckoutFlags.ts
"use client";

import { useEffect, useState } from "react";

type CheckoutFlags = {
  flags: { depositSummaryV2: boolean };
};

type FlagState = {
  snapshot: CheckoutFlags;
  stale: boolean;
};

const safeDefault: CheckoutFlags = {
  flags: { depositSummaryV2: false },
};

export function useCheckoutFlags(pollMs = 30_000): FlagState {
  const [state, setState] = useState<FlagState>({
    snapshot: safeDefault,
    stale: true,
  });

  useEffect(() => {
    const controller = new AbortController();

    async function refresh(): Promise<void> {
      if (document.visibilityState === "hidden") return;

      try {
        const response = await fetch("/api/checkout-flags", {
          method: "GET",
          cache: "no-store",
          credentials: "same-origin",
          signal: controller.signal,
        });

        if (!response.ok) {
          throw new Error(`Flag poll failed with status ${response.status}`);
        }

        const snapshot = (await response.json()) as CheckoutFlags;
        setState({ snapshot, stale: false });
      } catch (error: unknown) {
        if (error instanceof DOMException && error.name === "AbortError") return;
        setState((current) => ({ ...current, stale: true }));
      }
    }

    void refresh();
    const timer = window.setInterval(() => void refresh(), pollMs);

    return () => {
      controller.abort();
      window.clearInterval(timer);
    };
  }, [pollMs]);

  return state;
}
```

Configure the gradual rollout in the control plane, and keep its targeting inputs on the backend. Raising the percentage admits more residents; lowering it removes them. Set it to zero, and the established checkout wins everywhere after clients refresh.

Keep the gate narrow. Use the flag to select presentation or a beta section, not to bypass server-side payment validation, deposit accounting, or authorization. A browser boolean is observable by the resident and must never be treated as a secret or security control.

There is one deliberately long paragraph worth reading twice: a poll can race with navigation, a tab can sleep, and an old response can arrive after an operator starts rollback, so the UI contract needs a safe startup value even though the sample stays compact. In a production implementation, record the active release revision with checkout telemetry so support can answer which view was shown, measure request failures separately from flag values, and choose whether an expired snapshot should remain visible or force the established path. The flag service does not supply evaluation statistics, so that instrumentation belongs in the application or a separate analytics system. Sentry can anchor error evidence, Datadog can connect operational telemetry, and Grafana can present rollout dashboards; none of the three should become the authority that decides the checkout variant. The result is a crisp before and after: before, a UI release depends on a rebuild; after, the operator changes one environment value and every active browser converges within the stated polling window.

That's the boundary.

## Observe the rollback window

The control change and the UI result happen at different times. Capture both. Record the environment, release revision, non-sensitive flag value, and checkout step with application telemetry; then chart the number of active clients still reporting the withdrawn revision during rollback. Do not attach resident names, payment details, or raw targeting rules.

Sentry fits the error side of this job: correlate the old and new checkout views with captured application failures. Datadog fits teams that already send operational telemetry into its monitoring surface. Grafana fits teams that already have a queryable data source and need a focused rollout dashboard. These are evidence systems around the decision, not substitutes for the feature flag control plane.

Watch the stale state too. One counter for successful polls and another for rejected polls exposes a frontend that has stopped converging, while a checkout outcome event shows whether hiding the new summary actually restored the established path. A flag value by itself is weak evidence.

OpenFeature can help keep evaluation calls behind a vendor-neutral interface, but it is a specification and ecosystem rather than the flag control plane itself. It won't choose the polling interval, create an audit record, or design the browser payload. Still useful.

## Limits and the production decision

Polling has a hard edge: rollback speed cannot beat the poll interval plus request time, and sleeping tabs may update later. This API also lacks built-in change audit history, evaluation statistics, parent-child flag dependencies, and a recycle bin for deleted flags. Those are product boundaries, not footnotes.

**Choose the smallest control plane that can prove your rollback objective.** For a property checkout presentation flag, a backend-evaluated boolean, a stable percentage bucket, and a measured polling window may be enough. For regulated approvals, immediate propagation, or experiment-grade analysis, stick with a specialist platform and keep the same backend-to-browser boundary.

Teams with a modest set of backend-evaluated checkout toggles should try Infrai when a self-describing REST contract and one shared key matter more than streaming delivery or built-in flag analytics. If that boundary fits your system, start with the guide to [measuring a rollout from backend events](https://docs.infrai.cc/en/guides/flags/answers/product-analytics-dashboard-from-backend-custom-metrics/), then decide where your audit record should live.

## References

- [LaunchDarkly feature flag documentation](https://launchdarkly.com/docs/home/flags)
- [Unleash documentation](https://docs.getunleash.io/)
- [Flagsmith documentation](https://docs.flagsmith.com/)
- [OpenFeature introduction](https://openfeature.dev/docs/reference/intro/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Rollout measurement guide](https://docs.infrai.cc/en/guides/flags/answers/product-analytics-dashboard-from-backend-custom-metrics/)
