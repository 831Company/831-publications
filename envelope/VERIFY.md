# Verify this envelope

Layer 1 (before): run the classical verifier on each bundle with pins.v1.json; the lines
must equal index.json layers[0].expect. Layer 2 (arrival): run verify2 with pins.v2.json, keys-chain.json and
each H-seal; the lines must equal layers[1].expect (the same verdicts, plus assurance HYBRID). Every file is
listed in MANIFEST.sha256, whose sha256 is the published number's manifest_sha256.
