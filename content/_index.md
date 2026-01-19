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

<div class="hx:text-center hx:mb-8 hx:text-lg hx:text-gray-400">
Select the path that matches your role and start your journey
</div>

<div class="hx:mb-12 hx:grid hx:gap-6 hx:md:grid-cols-2">
  <div class="path-card">
    <h3>Algorithm Developer</h3>
    <p>Develop and evaluate root cause analysis algorithms using standardized datasets.</p>
    <ul class="hx:mb-4 hx:list-none hx:space-y-2">
      <li>Access standardized telemetry datasets</li>
      <li>Implement ML-based or rule-based RCA algorithms</li>
      <li>Evaluate locally with rcabench-platform</li>
      <li>Submit for remote evaluation via AegisLab</li>
    </ul>
    <a href="algorithm-developers" class="aegis-button">Start Developing Algorithms</a>
  </div>

  <div class="path-card">
    <h3>Dataset Creator</h3>
    <p>Create datasets through intelligent fault injection experiments in microservices systems.</p>
    <ul class="hx:mb-4 hx:list-none hx:space-y-2">
      <li>Execute fault injection via AegisLab API</li>
      <li>Use genetic algorithms for intelligent scheduling</li>
      <li>Collect traces, metrics, and logs automatically</li>
      <li>Generate standardized datasets for evaluation</li>
    </ul>
    <a href="dataset-creators" class="aegis-button">Start Creating Datasets</a>
  </div>
</div>

## Quick Links

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

<div class="hx:text-center hx:mb-8 hx:text-gray-400">
The AegisLab ecosystem consists of six interconnected components
</div>

<div class="ecosystem-grid hx:grid hx:gap-4 hx:grid-cols-1 hx:md:grid-cols-2 hx:lg:grid-cols-3 hx:mb-12">
  <div class="ecosystem-component">
    <div class="component-icon">⚡</div>
    <strong>AegisLab (RCABench)</strong>
    <span>Central orchestration platform for fault injection and RCA algorithm execution</span>
  </div>
  <div class="ecosystem-component">
    <div class="component-icon">🚂</div>
    <strong>TrainTicket</strong>
    <span>Target microservices application with 40+ Spring Boot services</span>
  </div>
  <div class="ecosystem-component">
    <div class="component-icon">💥</div>
    <strong>Chaos Experiment</strong>
    <span>Programmatic wrapper for Chaos Mesh with intelligent fault generation</span>
  </div>
  <div class="ecosystem-component">
    <div class="component-icon">📊</div>
    <strong>LoadGenerator</strong>
    <span>Realistic load generation with probabilistic behavior patterns</span>
  </div>
  <div class="ecosystem-component">
    <div class="component-icon">🧬</div>
    <strong>Pandora</strong>
    <span>Genetic algorithm-based intelligent fault injection scheduler</span>
  </div>
  <div class="ecosystem-component">
    <div class="component-icon">🔬</div>
    <strong>RCABench Platform</strong>
    <span>Algorithm development framework and evaluation toolkit</span>
  </div>
</div>

{{< github-contributors owner="OperationsPAI" repo="AegisLab" limit="30" >}}
