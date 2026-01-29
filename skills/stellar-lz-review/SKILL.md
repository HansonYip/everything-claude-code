---
name: stellar-lz-review
description: Reviews LayerZero cross-chain messaging code for security vulnerabilities specific to cross-chain protocols.
argument-hint: [contract-name-or-path]
---

# LayerZero Cross-Chain Security Review

Review the LayerZero protocol implementation at $ARGUMENTS for cross-chain specific vulnerabilities:

## Message Security

### 1. Message Replay Protection
- Verify nonce management prevents replay attacks
- Check that message GUIDs are unique and validated
- Ensure completed messages cannot be re-executed

### 2. Source Chain Validation
- Verify endpoint ID (EID) validation on receive
- Check that sender addresses are validated against trusted peers
- Ensure origin verification is complete before execution

### 3. Message Encoding/Decoding
- Check for length validation on encoded messages
- Look for buffer overflow possibilities
- Verify packet structure validation

## DVN/Verification Security

### 4. DVN Quorum Bypass
- Can messages be verified with fewer DVNs than required?
- Check threshold enforcement logic
- Verify signature counting is accurate

### 5. Signature Validation
- Check for signature malleability issues
- Verify no duplicate signers allowed
- Ensure signature recovery is correct

### 6. DVN Configuration Manipulation
- Can DVN config be changed for pending messages?
- Check config timing/ordering issues

## Fee/Economic Security

### 7. Fee Manipulation
- Can users underpay fees and still send?
- Check for fee calculation overflow
- Verify refund logic cannot be exploited

### 8. Native Token Handling
- Check for stuck funds scenarios
- Verify native drop limits
- Ensure fee collection is accurate

## Executor Security

### 9. Arbitrary Execution
- Verify executor cannot execute unauthorized calls
- Check callback validation
- Ensure proper gas limits

### 10. Message Ordering
- Can messages be delivered out of order?
- Check nonce enforcement
- Verify lazy nonce handling edge cases

## OApp Security

### 11. Peer Configuration
- How are trusted peers set/changed?
- Check for unauthorized peer changes
- Verify peer validation on send/receive

### 12. Composed Message Security
- Check lzCompose callback security
- Verify composed message ordering
- Look for reentrancy in compose flow

## Trust Assumptions

Document all trust assumptions:
- What happens if source chain is compromised?
- What if DVNs collude?
- What if executor is malicious?

## Output Format

For each finding, provide:
1. **Category**: Message/DVN/Fee/Executor/OApp
2. **Severity**: Critical/High/Medium/Low
3. **Location**: file:line
4. **Issue**: Clear description
5. **Attack Scenario**: How it could be exploited
6. **Remediation**: Recommended fix
