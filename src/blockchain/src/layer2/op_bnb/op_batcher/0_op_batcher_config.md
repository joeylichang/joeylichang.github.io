# op-batcher config

```go
type CLIConfig struct {
	MaxChannelDuration              : 32,
	SubSafetyMargin                 : 30,
	PollInterval                    : 10s,
	MaxPendingTransactions          : 25,
	MaxL1TxSize                     : 120,000,
	TargetNumFrames                 : 6,
	ApproxComprRatio                : 0.4,
	Compressor                      : "shadow",
	CompressionAlgo                 : "brotli",
	Stopped                         : false,
	WaitNodeSync                    : false,
	CheckRecentTxsDepth             : 0,
	BatchType                       : 1, // SpanBatch
	DataAvailabilityType            : "auto",
	TestUseMaxTxSizeForBlobs        : false,
	ActiveSequencerCheckDuration    : 2 * time.Minute,
}

type ChannelConfig struct {
    SeqWindowSize       : 14400 //(RollupConfig.SeqWindowSize).
    ChannelTimeout      : 1200  //(RollupConfig.ChannelTimeout).
    MaxChannelDuration  : 32,
    SubSafetyMargin     : 30,
    MaxFrameSize        : 120,000-1,
    TargetNumFrames     : 6,
    CompressorConfig,   : Config {
                                TargetOutputSize    : TargetNumFrames * MaxFrameSize = 6 * (120,000 - 1 - 23) ~ 720K,
                                ApproxComprRatio    : 0.4,
                                Kind                : "shadow",
                                CompressionAlgo     : "brotli",
                        }
    BatchType           : 1,    // SpanBatch
    MultiFrameTxs       : true  // BlobsType or AutoType
}
```