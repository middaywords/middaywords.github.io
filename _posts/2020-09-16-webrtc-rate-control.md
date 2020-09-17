---
title: webrtc, rate control
date: 2020-09-16
permalink: /posts/2020/09/webrtc/rate
tags:
  - webrtc
---

### webrtc rate control

When navigating through the code in `GoogCcNetworkControllerFactory`, I found that `OnNetworkStateEstimate()` is added.  

In webrtc(version m84), `NetworkStateEstimator` has been added. We can feed some transport information to the estimator and get some estimation about link capacity. 
While in the simulation for webrtc, we can use it to control the bitrate.



---

Definition for `NetworkStateEstimator`:

```c++
//api/transport/network_control
// Under development, subject to change without notice.
class NetworkStateEstimator {
 public:
  // Gets the current best estimate according to the estimator.
  virtual absl::optional<NetworkStateEstimate> GetCurrentEstimate() = 0;
  // Called with per packet feedback regarding receive time.
  // Used when the NetworkStateEstimator runs in the sending endpoint.
  virtual void OnTransportPacketsFeedback(const TransportPacketsFeedback&) = 0;
  // Called with per packet feedback regarding receive time.
  // Used when the NetworkStateEstimator runs in the receiving endpoint.
  virtual void OnReceivedPacket(const PacketResult&) {}
  // Called when the receiving or sending endpoint changes address.
  virtual void OnRouteChange(const NetworkRouteChange&) = 0;
  virtual ~NetworkStateEstimator() = default;
};

class NetworkStateEstimatorFactory {
 public:
  virtual std::unique_ptr<NetworkStateEstimator> Create(
      const WebRtcKeyValueConfig* key_value_config) = 0;
  virtual ~NetworkStateEstimatorFactory() = default;
};
```

As we can see, `OnReceivedPacket()` can tune the estimator in receive point, and send information back to sender. `OnTransportPacketsFeedback()` can tune when send point receives the feedback. And we can get `NetworkStateEstimate` when we need to use the estimated values.



---

Definition for `NetworkStateEstimate`, actually only `link_capacity`, `link_capacity_lower` and `link_capacity_upper` is used, other information is for debugging.

```c++
//api/transport/network_types.h
// Under development, subject to change without notice.
struct NetworkStateEstimate {
  double confidence = NAN;
  // The time the estimate was received/calculated.
  Timestamp update_time = Timestamp::MinusInfinity();
  Timestamp last_receive_time = Timestamp::MinusInfinity();
  Timestamp last_send_time = Timestamp::MinusInfinity();

  // Total estimated link capacity.
  DataRate link_capacity = DataRate::MinusInfinity();
  // Used as a safe measure of available capacity.
  DataRate link_capacity_lower = DataRate::MinusInfinity();
  // Used as limit for increasing bitrate.
  DataRate link_capacity_upper = DataRate::MinusInfinity();

  TimeDelta pre_link_buffer_delay = TimeDelta::MinusInfinity();
  TimeDelta post_link_buffer_delay = TimeDelta::MinusInfinity();
  TimeDelta propagation_delay = TimeDelta::MinusInfinity();

  // Only for debugging
  TimeDelta time_delta = TimeDelta::MinusInfinity();
  Timestamp last_feed_time = Timestamp::MinusInfinity();
  double cross_delay_rate = NAN;
  double spike_delay_rate = NAN;
  DataRate link_capacity_std_dev = DataRate::MinusInfinity();
  DataRate link_capacity_min = DataRate::MinusInfinity();
  double cross_traffic_ratio = NAN;
};
```



---

How it is used in receive point?

```c++
//modules/remote_bitrate_estimator/remote_estimator_proxy.cc
void RemoteEstimatorProxy::IncomingPacket(int64_t arrival_time_ms,
                                          size_t payload_size,
                                          const RTPHeader& header) {
  // ...
  
  if (network_state_estimator_ && header.extension.hasAbsoluteSendTime) {
    PacketResult packet_result;
    packet_result.receive_time = Timestamp::Millis(arrival_time_ms);
    // Ignore reordering of packets and assume they have approximately the same
    // send time.
    abs_send_timestamp_ += std::max(
        header.extension.GetAbsoluteSendTimeDelta(previous_abs_send_time_),
        TimeDelta::Millis(0));
    previous_abs_send_time_ = header.extension.absoluteSendTime;
    packet_result.sent_packet.send_time = abs_send_timestamp_;
    // TODO(webrtc:10742): Take IP header and transport overhead into account.
    packet_result.sent_packet.size =
        DataSize::Bytes(header.headerLength + payload_size);
    packet_result.sent_packet.sequence_number = seq;
    network_state_estimator_->OnReceivedPacket(packet_result);
  }
}

void RemoteEstimatorProxy::SendPeriodicFeedbacks() {
  // ...
  
  std::unique_ptr<rtcp::RemoteEstimate> remote_estimate;
  if (network_state_estimator_) {
    absl::optional<NetworkStateEstimate> state_estimate =
        network_state_estimator_->GetCurrentEstimate();
    if (state_estimate) {
      remote_estimate = std::make_unique<rtcp::RemoteEstimate>();
      remote_estimate->SetEstimate(state_estimate.value());
    }
  }
}
```



---

How it is used in send point?

```c++
//modules/congestion_controller/goog_cc/goog_cc_network_control.cc
NetworkControlUpdate GoogCcNetworkController::OnTransportPacketsFeedback(
    TransportPacketsFeedback report) {
  // ...
  
  if (network_estimator_) {
    network_estimator_->OnTransportPacketsFeedback(report);
    auto prev_estimate = estimate_;
    estimate_ = network_estimator_->GetCurrentEstimate();
    // TODO(srte): Make OnTransportPacketsFeedback signal whether the state
    // changed to avoid the need for this check.
    if (estimate_ && (!prev_estimate || estimate_->last_feed_time !=
                                            prev_estimate->last_feed_time)) {
      event_log_->Log(std::make_unique<RtcEventRemoteEstimate>(
          estimate_->link_capacity_lower, estimate_->link_capacity_upper));
    }
  }
  
  // ...
  
  DelayBasedBwe::Result result;
  result = delay_based_bwe_->IncomingPacketFeedbackVector(
      report, acknowledged_bitrate, probe_bitrate, estimate_,
      alr_start_time.has_value());
  
  // ...
}
NetworkControlUpdate GoogCcNetworkController::OnNetworkStateEstimate(
    NetworkStateEstimate msg) {
  estimate_ = msg;
  return NetworkControlUpdate();
}
```

`OnNetworkStateEstimate()` is to log the receive point's estimate.

Then we can jump into `delay_based_bwe_->IncomingPacketFeedbackVector`

```c++
//modules/congestion_controller/goog_cc/delay_based_bwe.cc
DelayBasedBwe::Result DelayBasedBwe::IncomingPacketFeedbackVector(
    const TransportPacketsFeedback& msg,
    absl::optional<DataRate> acked_bitrate,
    absl::optional<DataRate> probe_bitrate,
    absl::optional<NetworkStateEstimate> network_estimate,
    bool in_alr) {
  // code 
  
  // AimdRateControl rate_control_;
  rate_control_.SetNetworkStateEstimate(network_estimate);
  // std::move(network_estimate) is not used in MaybeUpdateEstimate()
  return MaybeUpdateEstimate(acked_bitrate, probe_bitrate,
                             std::move(network_estimate),
                             recovered_from_overuse, in_alr, msg.feedback_time);
}
```

Then we can see how `AimdRateControl` use the estimate

```c++
//modules/remote_bitrate_estimator/aimd_rate_control.cc
void AimdRateControl::SetNetworkStateEstimate(
    const absl::optional<NetworkStateEstimate>& estimate) {
  network_estimate_ = estimate;
}

void AimdRateControl::ChangeBitrate(const RateControlInput& input,
                                    Timestamp at_time) {
  // code 
  if (estimate_bounded_backoff_ && network_estimate_) {
    decreased_bitrate = std::max(
        decreased_bitrate, network_estimate_->link_capacity_lower * beta_);
  }
}

DataRate AimdRateControl::ClampBitrate(DataRate new_bitrate) const {
  if (estimate_bounded_increase_ && network_estimate_) {
    DataRate upper_bound = network_estimate_->link_capacity_upper;
    new_bitrate = std::min(new_bitrate, upper_bound);
  }
  new_bitrate = std::max(new_bitrate, min_configured_bitrate_);
  return new_bitrate;
}
```

We can see the video sending bitrate is bounded within [`network_estimate_->link_capacity_upper`, `network_estimate_->link_capacity_upper` ]. So we can easily control the bitrate using the estimator by setting these values.



---

How to use our own estimator?

In m84 version, there is no implementation of `NetworkStateEstimator` instance in source code. And in `GoogCcNetworkController`, it is not enabled by default.

```
struct GoogCcConfig {
  std::unique_ptr<NetworkStateEstimator> network_state_estimator = nullptr;
  std::unique_ptr<NetworkStatePredictor> network_state_predictor = nullptr;
  bool feedback_only = false;
};
```

We can implement one by ourselves, and set `GoogCcConfig::network_state_predictor`.


