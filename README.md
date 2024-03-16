AS OF RIGHT NOW THIS IS ONLY USEFUL FOR `IBS-M2` POOL SENSOR. It's a hacky approach to limitations with the tuya apigw repo.

# Why

This is all just a workaround to a breaking API change: https://github.com/tuya/tuya-device-sharing-sdk/issues/11

I'd ideally like to merge this PR: https://github.com/home-assistant/core/pull/113214
into the base Tuya integration, but the API change means the non-standard data points aren't available in home assistant.

# What
I forked an old version of the HA repo, using the legacy Tuya SDK.
