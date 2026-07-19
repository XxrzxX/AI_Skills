# Full‑Stack Security Hardening  
Security guidance for React + Node.js + Supabase + Vercel

This project provides a practical, real‑world security baseline for modern JavaScript stacks. It focuses on the highest‑impact risks seen in production incidents across React frontends, Node.js APIs, Supabase Postgres, Vercel deployments, and GitHub Actions CI/CD.

## What’s inside
- **OWASP‑mapped application security guidance** for Node.js + React  
- **Supabase security notes** (RLS, anon/service_role keys, storage, auth)  
- **CI/CD & supply‑chain hardening** for GitHub Actions and npm  
- **Compliance crosswalks** for NCA ECC, SAMA CSF, ISO 27001/27002  
- **Incident‑response playbook** and secret‑rotation procedures  
- **Real case studies** (tj-actions compromise, Shai‑Hulud npm worm, RLS breaches)

## Why this exists
Most breaches on this stack come from shared‑responsibility gaps — missing RLS, leaked service_role keys, unpinned GitHub Actions, compromised npm packages, or preview deployments exposing secrets. This repo consolidates the controls that actually prevent those failures.

## Who this is for
Developers, founders, and teams preparing for production launch, security reviews, or compliance assessments.

## How to use it
Start with the “Eight Failure Modes” section, then apply the relevant checklists to your code, CI/CD, and Supabase/Vercel configuration. Use the incident‑response playbook to prepare for secret leaks or supply‑chain events.

