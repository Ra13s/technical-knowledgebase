# Externalize authoritative agent telemetry

Keep authoritative model/tool/executor telemetry in a control-plane component separate from the agent task workspace. Use stable run IDs, record executor outcomes, keep recorder credentials outside the runtime, apply privacy redaction before storage, and periodically test that completed runs can be reconstructed from control-plane telemetry alone.
