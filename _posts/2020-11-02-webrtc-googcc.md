---
title: notes for googcc in webrtc 
date: 2020-11-02
permalink: /posts/2020/11/gcc-webrtc/
tags:
  - webrtc
  - googcc
typora-root-url: ../../middaywords.github.io
---



### GoogCc 

The CC algorithm is located in `modules/congestion_controller/goog_cc`, we can have a look at `goog_cc_network_control.*` first.

In `goog_cc_network_control.*`, it defines `GoogCcNetworkController`, which extends `NetworkControllerInterface`  and contains many other tools.

```c++
class GoogCcNetworkController : public NetworkControllerInterface {
 public:
  // ...
  // NetworkControllerInterface
  NetworkControlUpdate OnNetworkAvailability(NetworkAvailability msg) override;
  NetworkControlUpdate OnNetworkRouteChange(NetworkRouteChange msg) override;
  // Called periodically with a periodicy as specified by
  // NetworkControllerFactoryInterface::GetProcessInterval. 
  // default value is 25ms
  NetworkControlUpdate OnProcessInterval(ProcessInterval msg) override;
  NetworkControlUpdate OnRemoteBitrateReport(RemoteBitrateReport msg) override;
  NetworkControlUpdate OnRoundTripTimeUpdate(RoundTripTimeUpdate msg) override;
  NetworkControlUpdate OnSentPacket(SentPacket msg) override;
  NetworkControlUpdate OnReceivedPacket(ReceivedPacket msg) override;
  NetworkControlUpdate OnStreamsConfig(StreamsConfig msg) override;
  NetworkControlUpdate OnTargetRateConstraints(
      TargetRateConstraints msg) override;
  NetworkControlUpdate OnTransportLossReport(TransportLossReport msg) override;
  NetworkControlUpdate OnTransportPacketsFeedback(
      TransportPacketsFeedback msg) override;
  NetworkControlUpdate OnNetworkStateEstimate(
      NetworkStateEstimate msg) override;

  NetworkControlUpdate GetNetworkState(Timestamp at_time) const;
  // ...
 private:
  // ...
  const std::unique_ptr<ProbeController> probe_controller_;
  const std::unique_ptr<CongestionWindowPushbackController>
      congestion_window_pushback_controller_;

  std::unique_ptr<SendSideBandwidthEstimation> bandwidth_estimation_;
  std::unique_ptr<AlrDetector> alr_detector_;
  std::unique_ptr<ProbeBitrateEstimator> probe_bitrate_estimator_;
  std::unique_ptr<NetworkStateEstimator> network_estimator_;
  std::unique_ptr<NetworkStatePredictor> network_state_predictor_;
  std::unique_ptr<DelayBasedBwe> delay_based_bwe_;
  std::unique_ptr<AcknowledgedBitrateEstimatorInterface>
      acknowledged_bitrate_estimator_;

  absl::optional<NetworkControllerConfig> initial_config_;
  // ...
}
```

* NetwrokController: NetworkControllerInterface is implemented by network controllers. A network controller is a class that uses information about network state and traffic to estimate network parameters such as round trip time and bandwidth. Network controllers does not guarantee thread safety, the interface must be used in a non-concurrent fashion.
* ProbeController

Architecture:

<img src="/_posts/assets/image-20201102173342641.png" alt="image-20201102173342641" style="zoom:50%;" />



Reference 

* rmcat gcc doc : https://tools.ietf.org/html/draft-ietf-rmcat-gcc-02
* MMSys paper : https://www.aitrans.online/static/paper/Gcc-analysis.pdf
* RTP protocol doc: https://tools.ietf.org/html/rfc3550
* Blog notes: https://www.jianshu.com/p/0f7ee0e0b3be 