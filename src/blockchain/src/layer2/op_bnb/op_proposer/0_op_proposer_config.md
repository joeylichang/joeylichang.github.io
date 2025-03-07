# op-proposer config

```go
type CLIConfig struct {
	PollInterval                    : 6s,
	AllowNonFinalized               : false,
	ProposalInterval                : 0, // when the dispute game factory address is set
	ActiveSequencerCheckDuration    : 2 * time.Minute,
	WaitNodeSync                    : false,
}
```