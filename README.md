# RoleRadar

RoleRadar is a configurable automation system for monitoring targeted
companies and detecting newly posted or updated roles using
authoritative applicant tracking system (ATS) data sources. It is
designed to surface high-signal opportunities with minimal noise through
scheduled scans, normalization, deduplication, and rule-based filtering.

Built on **n8n**, RoleRadar uses deterministic workflows and modular
adapters to ingest job data directly from ATS platforms (such as
Greenhouse, Workday, and Lever), avoiding reliance on broad job board
scraping. The system is portable by design and can be run locally via
Docker or adapted for hosted deployments.

------------------------------------------------------------------------

## Key Capabilities

-   Targeted company tracking (explicit company lists, not keyword
    scraping)
-   ATS-native data ingestion for accuracy and freshness
-   Canonical job normalization across multiple ATS providers
-   Stateful deduplication to detect net-new or updated roles
-   Configurable matching rules (e.g., role titles, seniority, domains)
-   Daily or scheduled reporting via email or other notification
    channels
-   Modular architecture suitable for extension or productization

------------------------------------------------------------------------

## High-Level Architecture

RoleRadar is organized around a small set of composable workflows:

-   **Orchestrator Workflow**\
    Coordinates scheduled runs, loads configuration, routes companies to
    the appropriate ATS adapter, performs deduplication and filtering,
    and generates reports.

-   **ATS Adapter Workflows**\
    One adapter per ATS platform (e.g., Greenhouse, Workday),
    responsible for fetching and normalizing job data into a canonical
    schema.

-   **State Management Layer**\
    Tracks companies and previously seen jobs to enable accurate change
    detection. Can be backed by n8n data tables during prototyping or
    PostgreSQL for production use.

------------------------------------------------------------------------

## Configuration Model

RoleRadar is driven by a centralized configuration file that defines:

-   The list of companies to monitor and their careers URLs
-   Role-matching rules (e.g., regular expressions for titles/seniority)
-   Reporting and notification settings
-   Runtime options such as batching and scan frequency

This design allows multiple job seekers or deployments to share the same
workflows while using different targeting rules.

------------------------------------------------------------------------

## Deployment

RoleRadar supports two primary modes:

-   **Cloud Development**\
    Designed and tested using n8n Cloud, with configuration loaded
    remotely (e.g., via GitHub-hosted config files).

-   **Local Deployment**\
    Deployed locally using Docker (n8n + PostgreSQL), enabling full
    control over state, credentials, and execution environment.

A clear migration path exists from cloud-based prototyping to local or
self-hosted execution.

------------------------------------------------------------------------

## Intended Use

RoleRadar is suitable for: - Personal, high-signal job monitoring -
Shared or team-based role tracking - Experimentation with structured
job-market intelligence - Future open-source or commercial extensions

It is not intended to replace job boards, but to provide a precise,
low-noise complement to them.

------------------------------------------------------------------------

## Status

This project is under active development and is currently configured for
private use. Public release, licensing, and contribution guidelines may
be defined in the future.
