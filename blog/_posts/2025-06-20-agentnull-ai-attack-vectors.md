---
layout: post
title: "ThirdKey's AgentNull: Unveiling the Growing Catalog of AI Attack Vectors"
date: 2025-06-20
categories: [AI Security, Research, AgentNull]
tags: [AI, security, attack vectors, MCP, RAG, vector databases, autonomous agents]
excerpt: "Exploring ThirdKey's comprehensive catalog of AI attack vectors targeting autonomous agents, RAG pipelines, and vector databases - complete with proof-of-concept demonstrations."
---

The age of autonomous AI agents is upon us, and with it comes a new frontier of security challenges. As these agents become more integrated into our digital lives, understanding and mitigating their potential vulnerabilities is paramount. This is where ThirdKey's AgentNull project becomes an invaluable resource for the cybersecurity community.

AgentNull, a project by ThirdKey Research, is a comprehensive, red-team-oriented catalog of attack vectors that target a wide range of AI systems, from autonomous agents and RAG pipelines to vector databases and embedding-based retrieval systems. Each attack vector is accompanied by a proof-of-concept (PoC), allowing researchers and developers to understand and replicate these vulnerabilities in a controlled environment.

This blog post will explore some of the key attack categories covered in the AgentNull catalog, highlighting the innovative research being done by ThirdKey to secure the next generation of AI.

## A Multitude of Attack Vectors

The AgentNull catalog is extensive, covering a wide array of vulnerabilities. Here are some of the key areas of focus:

### MCP & Agent Systems

This is a major focus of the catalog, with a number of novel attacks, including:

* **Full-Schema Poisoning (FSP):** This attack goes beyond traditional tool poisoning by exploiting any field in an MCP tool schema, not just the description. For example, a parameter could be maliciously named `content_from_reading_ssh_id_rsa` to trick the LLM into accessing sensitive files.

* **Advanced Tool Poisoning Attack (ATPA):** This technique manipulates tool *outputs* to trigger secondary malicious actions. For instance, a tool could return a fake error message that requests sensitive data.

* **MCP Rug Pull Attack:** This attack exploits the trust between developers and MCP servers by swapping benign tool descriptions with malicious ones after the tool has been approved for production.

* **Schema Validation Bypass:** Attackers can exploit inconsistencies in how different MCP clients validate tool schemas, allowing them to craft payloads that bypass some validators while being accepted by others.

### Memory & Context Systems

These attacks manipulate the agent's memory and context to bypass safety measures:

* **Recursive Leakage:** Sensitive information can be summarized and leak into later, unrelated messages.

* **Token Gaslighting:** This involves flooding the agent's memory with junk data to push out earlier safety instructions.

### RAG & Vector Systems

These attacks focus on the vulnerabilities of Retrieval-Augmented Generation and vector database systems:

* **Cross-Embedding Poisoning:** This attack manipulates vector embeddings to make malicious content appear more similar to legitimate content, increasing the likelihood of it being retrieved.

* **Index Skew Attacks:** This theoretical attack involves biasing vector database indexing mechanisms to favor the retrieval of malicious content.

## The Importance of Proactive Security Research

The work being done by ThirdKey's AgentNull project is a critical component of a proactive cybersecurity strategy. By identifying and documenting these vulnerabilities before they are widely exploited, the security community can develop the necessary defenses to protect against them. The detailed PoCs provided in the AgentNull repository are an invaluable tool for researchers, developers, and security professionals who are working to build a more secure AI ecosystem.

As AI continues to evolve, so too will the methods used to attack it. ThirdKey's AgentNull project is essential for staying ahead of the curve and ensuring that the next generation of AI is both powerful and secure.

---

**Learn More:** Explore the complete AgentNull catalog and access proof-of-concept demonstrations at the [AgentNull GitHub repository](https://github.com/jaschadub/AgentNull).