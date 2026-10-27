# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
47
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->256.2<!--/answer--> | <!--answer:vm_warm_median_ms-->212.1<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2276.8<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->130.6<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1364.8<!--/answer--> | <!--answer:function_warm_median_ms-->134.9<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
[OK  ] vm        cold=  256.2ms  warm_median=  212.1ms
            ########################################
[OK  ] cloudrun  cold= 2276.8ms  warm_median=  130.6ms
            #########################
[OK  ] function  cold= 1364.8ms  warm_median=  134.9ms
            #########################
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
The VM creates nothing extra: one e2-micro handles every request, so ten or a hundred concurrent clients queue on that box. The hard limit is the machine I picked (`--machine-type=e2-micro`). Cloud Run starts more container instances as load rises, up to the flag I set (`--max-instances=3`); beyond that, requests wait or fail. The gen-2 Function is the same autoscaler (it is a Cloud Run service) with `--min-instances=0`, so it scales to zero when idle and spins instances under load, again capped by Cloud Run.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
The VM: a custom VPC (`echo-net` / `echo-subnet`), tagged firewall rules for 22 and 8080, Artifact Registry Reader on the instance SA, an `echo-vm` with `--tags=echo-http`, then SSH, install Docker, authenticate, and `docker run`. I also have to keep the VM and its external IP up. Cloud Run was one `gcloud run deploy` of the same `echo-bot:v1` image, port 8080, min 0 / max 3, public. The Function did not use that image: `gcloud functions deploy --gen2` uploaded `phase-1-echo-bot` source (`main.py` / `app.py`) and Cloud Build packed it.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-208953144909.us-central1.run.app
LAST DEPLOYED BY: service-208953144909@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-27T13:07:32.049356Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-208953144909.us-central1.run.app
LAST DEPLOYED BY: riana.rakotovao@eurecom.fr
LAST DEPLOYED AT: 2026-10-27T11:29:15.787691Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
`gcloud run services list` shows two Cloud Run services in `us-central1`: `echo-bot` and `echo`. I deployed `echo-bot` from the prebuilt `echo-bot:v1` image. `echo` was last deployed by `service-208953144909@gcf-admin-robot.iam.gserviceaccount.com`, the Cloud Functions robot. So a gen-2 Function is not a separate runtime: Cloud Functions is a front-end that creates and manages a Cloud Run service. That is why the warm medians match (130.6 ms vs 134.9 ms) and why both have a real cold start after scale-to-zero (2276.8 ms and 1364.8 ms), unlike the VM (256.2 vs 212.1). The real difference is the deploy path: Cloud Run runs the image I pushed; the Function uploads source and buildpacks build an image. URLs and the deployer identity differ; the serving platform does not.
<!--/answer-->
