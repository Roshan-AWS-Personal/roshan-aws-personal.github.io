---
layout: default
title: Introduction
---

Hello and welcome to my first project! This page is the first tangible demonstration of marrying my professional software engineering experience with my newly acquired AWS Solutions Architect knowledge. What started as a simple file-upload API—just a Lambda function writing to an S3 bucket—grew into a fully distributed, best-practice architecture template that any client-branded UI can plug into.
My goal was twofold: apply the theoretical concepts from my AWS certification in a real-world context, and bring in the rigorous engineering practices I’ve honed over my career. The result is a resilient web front end served from Amazon S3 and accelerated by CloudFront, secured by Amazon Cognito’s OAuth 2.0 flow, and underpinned by Lambda functions behind API Gateway, DynamoDB for event-driven metadata capture, and SES for notifications. While the static UI is intentionally generic—designed to be swapped out for client-specific branding—the surrounding infrastructure remains rock-solid.
Equally important is how I manage this framework. Every AWS resource lives in Terraform modules, with clear dev/prod isolation and configuration driven by GitHub secrets. Pull-request–gated terraform plan checks, protected branches, and click-to-deploy GitHub Actions workflows enforce reproducibility, safety, and collaborative best practices—just like a seasoned enterprise team.
In the chapters that follow, we’ll trace the evolution of this project through key principles:
1.	Infrastructure as Code: How a console-click prototype became Terraform-managed, multi-environment code.
2.	Authentication & Security: Transitioning from basic tokens to Cognito OAuth for enterprise-grade access control.
3.	Scalable Data Flows: Migrating from direct Lambda uploads to pre-signed URLs to optimize performance and cost.
4.	CORS & CDN Strategies: Overcoming cross-domain challenges with CloudFront and precise header management.
5.	Event-Driven Logging & Notifications: Decoupling workflows with DynamoDB streams and SES alerts.
Each section details the challenge I encountered, the solution I implemented, and the lesson that solidified my understanding of AWS architectures in a professional setting.

