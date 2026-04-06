# 🏛️ Software Engineering & Architecture Vault

Welcome to my personal technical vault. This repository is a curated collection of Architectural Decision Records (ADRs), Performance Spikes, and Technical Case Studies derived from over 10 years of experience leading high-scale enterprise applications.

The goal of this vault is to demonstrate not just how I code, but how I think and solve complex engineering challenges at scale.

## 🚀 Featured Case Studies

### 🏗️ System Design: Venti SaaS Architecture

**Problem:** Need for a highly scalable, multi-tenant business management platform with strict data isolation.

**Solution:** Engineered a modular architecture using NestJS and React 19, implementing Row-Level Security (RLS) and tenant-specific contexts.

**Impact:** A production-ready blueprint for secure, concurrent enterprise operations and real-time communications.

### 🍽️ System Design: miPOS — Restaurant OS

**Problem:** High operational friction in the restaurant industry due to siloed delivery apps (Rappi/Uber Eats), isolated POS terminals, and manual inventory sync.

**Solution:** Engineered an API-first "Operating System" console using Next.js and a custom Node.js server to normalize fragmented data into a single, resilient omnichannel interface.

**Impact:** Eliminated manual data re-entry, saving 2–5 minutes per ticket and providing an open API for third-party marketplace integrations.

**Reference:** [View Implementation Case Study](./frontend-architecture/mipos-restaurant-os/) | [Source Code](https://github.com/rennyluzardo/miPOS)

### 📱 Mobile Engineering: Legacy Resilience

**Problem:** Maintaining high-availability (99.8% uptime) in a mission-critical logistics app running on legacy React Native 0.59.

**Solution:** Strategic implementation of dependency bridges (jetifier), manual native patches, and a phased 4-stage modernization roadmap.

**Impact:** Zero-downtime maintenance of industrial operations while eliminating critical security vulnerabilities.

### ⚡ Frontend: High-Performance Bundle Strategy

**Problem:** Monolithic enterprise platform with a >2MB initial bundle size affecting Core Web Vitals.

**Solution:** Implemented granular code-splitting, tree-shaking optimization, and a dynamic icon registry using Vite 7.

**Impact:** Projected reduction of 60% in initial load time and a +40 point increase in Lighthouse scores.

### 🎨 Frontend: Gestalt UI & Accessible Design

**Problem:** High cognitive load and visual fatigue in data-dense administrative dashboards.

**Solution:** Theme system based on Gestalt Psychology principles and WCAG 2.1 AAA contrast standards.

**Impact:** 6.5:1 contrast ratios and optimized information density for power users.

## 🛠️ Tech Stack & Patterns

**Frontend:** React 19, Vite 7, TypeScript, Redux Toolkit.

**Backend & System:** NestJS 11, PostgreSQL (RLS), Multi-tenant Architecture.

**Mobile:** React Native (Legacy & Modern), SRE Monitoring Patterns.

**Methodologies:** Atomic Design, Module Federation, Compound Components.

## 🔒 Confidentiality & Ethics

All project names, component identifiers, and specific business logic have been fully anonymized and abstracted to protect the intellectual property of past clients and proprietary ventures. These documents represent my personal technical methodology and architectural research.

## 📫 Connect with me

**LinkedIn:** linkedin.com/in/rennyluzardo

**Role:** Senior Frontend Engineer | Technical Lead

**Focus:** High-Performance Architectures & Scalable Systems