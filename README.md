def index_case(self, report: dict, patient_id: str) -> dict:
    text = self._report_to_text(report)
    chunks = self._chunk_text(text)
    embeddings = self.embeddings.embed_documents(chunks)

    collection = self.client.get_or_create_collection(self.collection_name)
    report_id = report.get("report_id", "x")

    collection.add(
        ids=[f"{patient_id}-{report_id}-{i}" for i in range(len(chunks))],
        embeddings=embeddings,
        metadatas=[{"patient_id": patient_id, "report_id": report_id, "chunk_id": i} for i in range(len(chunks))],
        documents=chunks,
    )

    return {
        "chunks_indexed": len(chunks),
        "collection_name": self.collection_name,
        "patient_id": patient_id,
    }

    def _chunk_text(self, text: str) -> List[str]:
    chunk_size = self.config.get("chunk_size", 256)
    chunk_overlap = self.config.get("chunk_overlap", 64)
    paragraphs = [p.strip() for p in re.split(r"\n\s*\n", text) if p.strip()]

    chunks: List[str] = []

    for paragraph in paragraphs:
        if len(paragraph) <= chunk_size:
            chunks.append(paragraph)
            continue

        step = max(1, chunk_size - chunk_overlap)
        for start in range(0, len(paragraph), step):
            window = paragraph[start:start + chunk_size]
            if window:
                chunks.append(window)
            if start + chunk_size >= len(paragraph):
                break

    return chunks

    def _report_to_text(self, report: dict) -> str:
    eligibility = report.get("eligibility_result", {}) or {}
    risk = report.get("risk_assessment", {}) or {}
    decision = report.get("decision", {}) or {}
    reflection = report.get("reflection", {}) or {}

    lines = [
        f"Report {report.get('report_id', 'unknown')} for patient {report.get('patient_id', 'unknown')} "
        f"({report.get('patient_name', 'unknown')}).",
        f"Requested service: {report.get('requested_service', 'unknown')} "
        f"(procedure code {report.get('procedure_code', 'unknown')}).",
        "",
        "Eligibility:",
        f"- Active coverage: {eligibility.get('is_active', 'unknown')} under {eligibility.get('plan_name', 'unknown')}.",
        f"- Benefit coverage: {eligibility.get('benefit_coverage_percent', 'unknown')}% of "
        f"standard cost {eligibility.get('standard_cost', 'unknown')}.",
        f"- Patient responsibility estimate: {eligibility.get('patient_responsibility_estimate', 'unknown')}.",
        f"- Eligible: {eligibility.get('eligible', 'unknown')} -- {eligibility.get('eligibility_reason', '')}",
        "",
        "Risk assessment:",
        f"- Risk level: {risk.get('risk_level', 'unknown')} (risk_score {risk.get('risk_score', 'unknown')}).",
        f"- Fraud indicators: {', '.join(risk.get('fraud_indicators', []) or []) or 'none'}.",
        f"- Requires peer review: {risk.get('requires_peer_review', 'unknown')}.",
        "",
        "Decision:",
        f"- Determination: {decision.get('decision', 'unknown')} (confidence {decision.get('confidence_score', 'unknown')}).",
        f"- Explanation: {decision.get('explanation', '')}",
        f"- Key findings: {'; '.join(decision.get('key_findings', []) or []) or 'none'}.",
        f"- Next steps: {'; '.join(decision.get('next_steps', []) or []) or 'none'}.",
        "",
        "Reflection (quality self-check):",
        f"- Groundedness: {reflection.get('groundedness', 'unknown')}, Relevance: {reflection.get('relevance', 'unknown')}, "
        f"Confidence alignment: {reflection.get('confidence_alignment', 'unknown')}.",
        f"- Passed: {reflection.get('passed', 'unknown')} -- {reflection.get('feedback', '')}",
        f"\nGenerated at: {report.get('generated_at', 'unknown')}",
    ]
    return "\n".join(lines)
