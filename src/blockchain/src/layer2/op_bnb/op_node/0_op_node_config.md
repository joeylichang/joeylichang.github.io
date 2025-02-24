# op-node config

```go
type Config struct {
    L1EpochPollInterval: 3s, // 周期获取 L1 事件周期，比如 safe，unsafe，finalize 的周期
    Driver: &driver.Config{
                VerifierConfDepth:       15,
                SequencerConfDepth:      15,
                SequencerEnabled:        true/false, // sequencer/fullnode
                SequencerStopped:        false/true, // sequencer/fullnode
                SequencerMaxSafeLag:     0, // disable
                SequencerPriority:       true,
                SequencerCombinedEngine: true,
            },
    Rollup: &rollup.Config{
        Genesis: rollup.Genesis{
                L1: eth.BlockID{
                Hash:   common.HexToHash("0x29443b21507894febe7700f7c5cd3569cc8bf1ba535df0489276d8004af81044"),
                Number: 30758357,
            },
            L2: eth.BlockID{
                Hash:   common.HexToHash("0x4dd61178c8b0f01670c231597e7bcb368e84545acd46d940a896d6a791dd6df4"),
                Number: 0,
            },
            L2Time: 1691753723,
            SystemConfig: eth.SystemConfig{
                BatcherAddr: common.HexToAddress("0xef8783382ef80ec23b66c43575a6103deca909c3"),
                Overhead:    eth.Bytes32(common.HexToHash("0x0000000000000000000000000000000000000000000000000000000000000834")),
                Scalar:      eth.Bytes32(common.HexToHash("0x00000000000000000000000000000000000000000000000000000000000f4240")),
                GasLimit:    100000000,
            },
        },
        BlockTime:              1,
        MaxSequencerDrift:      600,                // FjordTime 之后失效了
        SeqWindowSize:          14400,
        ChannelTimeout:         1200,
        L1ChainID:              big.NewInt(56),
        L2ChainID:              big.NewInt(204),
        BatchInboxAddress:      common.HexToAddress("0xff00000000000000000000000000000000000204"),
        DepositContractAddress: common.HexToAddress("0x1876ea7702c0ad0c6a2ae6036de7733edfbca519"),
        L1SystemConfigAddress:  common.HexToAddress("0x7ac836148c14c74086d57f7828f2d065672db3b8"),
        RegolithTime:           u64Ptr(0),
        Fermat:                 big.NewInt(9397477), // Nov-28-2023 06 AM +UTC
        SnowTime:               u64Ptr(1713160800),  // Apr-15-2024 06 AM +UTC
        CanyonTime:             u64Ptr(1718870400),  // Jun-20-2024 08:00 AM +UTC
        DeltaTime:              u64Ptr(1718871000),  // Jun-20-2024 08:10 AM +UTC
        EcotoneTime:            u64Ptr(1718871600),  // Jun-20-2024 08:20 AM +UTC
        FjordTime:              u64Ptr(1727157600),  // Sep-24-2024 06:00 AM +UTC
    },
}
```