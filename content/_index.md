---
title: AegisLab Documentation
layout: hextra-home
---

{{< hextra/hero-badge >}}
  <div class="hx:w-2 hx:h-2 hx:rounded-full hx:bg-primary-400"></div>
  <span>Fault Injection & RCA Evaluation Platform</span>
  {{< icon name="arrow-circle-right" attributes="height=14" >}}
{{< /hextra/hero-badge >}}

<div class="hx:mt-6 hx:mb-6">
{{< hextra/hero-headline >}}
  AegisLab Ecosystem&nbsp;<br class="hx:sm:block hx:hidden" />for Microservices Research
{{< /hextra/hero-headline >}}
</div>

<div class="hx:mb-12">
{{< hextra/hero-subtitle >}}
  Comprehensive platform for fault injection, chaos engineering,&nbsp;<br class="hx:sm:block hx:hidden" />and root cause analysis algorithm evaluation
{{< /hextra/hero-subtitle >}}
</div>

## Choose Your Path

Select the path that matches your role:

<div class="hx:mb-6 hx:grid hx:gap-4 hx:md:grid-cols-2">
  <div class="hx:rounded-lg hx:border hx:border-gray-200 hx:p-6 hx:dark:border-gray-800">
    <h3 class="hx:text-xl hx:font-bold hx:mb-2">Algorithm Developer</h3>
    <p class="hx:mb-4">Develop and evaluate root cause analysis algorithms using standardized datasets.</p>
    <ul class="hx:mb-4 hx:list-disc hx:list-inside hx:space-y-1">
      <li>Access standardized trace datasets</li>
      <li>Implement ML-based or rule-based RCA algorithms</li>
      <li>Evaluate locally with rcabench-platform</li>
      <li>Submit for remote evaluation via AegisLab</li>
    </ul>
    {{< hextra/hero-button text="Start Developing Algorithms" link="algorithm-developers" >}}
  </div>

  <div class="hx:rounded-lg hx:border hx:border-gray-200 hx:p-6 hx:dark:border-gray-800">
    <h3 class="hx:text-xl hx:font-bold hx:mb-2">Dataset Creator</h3>
    <p class="hx:mb-4">Create datasets through intelligent fault injection experiments in microservices systems.</p>
    <ul class="hx:mb-4 hx:list-disc hx:list-inside hx:space-y-1">
      <li>Execute fault injection via AegisLab API</li>
      <li>Use genetic algorithms for intelligent scheduling</li>
      <li>Collect traces, metrics, and logs automatically</li>
      <li>Generate standardized datasets for evaluation</li>
    </ul>
    {{< hextra/hero-button text="Start Creating Datasets" link="dataset-creators" >}}
  </div>
</div>

## Quick Links

<div class="hx:mt-6"></div>

{{< hextra/feature-grid >}}
  {{< hextra/feature-card
    title="Getting Started"
    subtitle="Understand the ecosystem architecture and choose your path"
    link="getting-started"
    icon="play"
  >}}
  {{< hextra/feature-card
    title="Deployment Guide"
    subtitle="Deploy AegisLab, TrainTicket, and supporting infrastructure"
    link="deployment"
    icon="document"
  >}}
  {{< hextra/feature-card
    title="Core Concepts"
    subtitle="Learn about fault injection, observability, and RCA evaluation"
    link="concepts"
    icon="book-open"
  >}}
  {{< hextra/feature-card
    title="Contributing"
    subtitle="Contribute to the AegisLab ecosystem"
    link="contributing"
    icon="sparkles"
  >}}
{{< /hextra/feature-grid >}}

## Ecosystem Components

The AegisLab ecosystem consists of six interconnected components:

- **AegisLab (RCABench)**: Central orchestration platform for fault injection and RCA algorithm execution
- **TrainTicket**: Target microservices application with 40+ Spring Boot services
- **Chaos Experiment**: Programmatic wrapper for Chaos Mesh with intelligent fault generation
- **LoadGenerator**: Realistic load generation with probabilistic behavior patterns
- **Pandora**: Genetic algorithm-based intelligent fault injection scheduler
- **RCABench Platform**: Algorithm development framework and evaluation toolkit

{{< github-contributors owner="OperationsPAI" repo="AegisLab" limit="30" >}}
