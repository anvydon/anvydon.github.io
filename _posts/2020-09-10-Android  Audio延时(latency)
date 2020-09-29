---
layout:     post
title:      Android Audio延时(latency)
subtitle:   Android Audio延时(latency)
date:       2020-09-10
author:     Joey
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Audio
---

## Android Audio 延迟 (latency)

>> 音频延时是音频信号通过系统是的时间延迟
>> 影响因素: 应用, 通道缓冲区总数, 缓冲区大小, 处理器

### AudioTrack播放延时

* AudioTrack类中latency()获取

```
    /* Returns this track's estimated latency in milliseconds.
     * This includes the latency due to AudioTrack buffer size, AudioMixer (if any)
     * and audio hardware driver.
     */
            uint32_t    latency() const     { return mLatency; }

```

* mLatency在createTrack_l中赋值, 其中afLatency是底层延时时间, (1000*frameCount) / mSampleRate Track buffer计算延时时间.

```
    status = AudioSystem::getLatency(output, &afLatency);

    mLatency = afLatency + (1000*frameCount) / mSampleRate;
```

* AudioSystem::getLatency 获取af->latency(output) 或者 outputDesc->latency

```
status_t AudioSystem::getLatency(audio_io_handle_t output,
                                 uint32_t* latency)
{
    const sp<IAudioFlinger>& af = AudioSystem::get_audio_flinger();
    if (af == 0) return PERMISSION_DENIED;

    Mutex::Autolock _l(gLockCache);

/* outputDesc 通过　audioConfigChanged　通知增加到　AudioSystem::gOutputs　全局向量中　*/
    OutputDescriptor *outputDesc = AudioSystem::gOutputs.valueFor(output);
    if (outputDesc == NULL) {
        gLockCache.unlock();
        *latency = af->latency(output);
        gLockCache.lock();
    } else {
        *latency = outputDesc->latency;
    }

    ALOGV("getLatency() output %d, latency %d", output, *latency);

    return NO_ERROR;
}
```

* af->latency(output) AudioFlinger获取thread->latency

```
uint32_t AudioFlinger::latency(audio_io_handle_t output) const
{
    Mutex::Autolock _l(mLock);
    PlaybackThread *thread = checkPlaybackThread_l(output);
    if (thread == NULL) {
        ALOGW("latency(): no playback thread found for output handle %d", output);
        return 0;
    }
    return thread->latency();
}
```

* thead->latency()获取底层hal latency

```
uint32_t AudioFlinger::PlaybackThread::latency() const
{
    Mutex::Autolock _l(mLock);
    return latency_l();
}
uint32_t AudioFlinger::PlaybackThread::latency_l() const
{
    if (initCheck() == NO_ERROR) {
        return correctLatency_l(mOutput->stream->get_latency(mOutput->stream));
    } else {
        return 0;
    }
}
```

* hal latency获取AudioALSAStreamOut::latency() 通过buffer计算延时

```
    static uint32_t out_get_latency(const struct audio_stream_out *stream)
    {
        const struct legacy_stream_out *out =
            reinterpret_cast<const struct legacy_stream_out *>(stream);
        return out->legacy_out->latency();
    }
    
    uint32_t AudioALSAStreamOut::latency() const
{
    uint32_t latency = 0;

    if (mPlaybackHandler == NULL)
    {
        latency = mStreamAttributeSource.latency;
    }
    else
    {
        const stream_attribute_t *pStreamAttributeTarget = mPlaybackHandler->getStreamAttributeTarget();
        const uint8_t size_per_channel = (pStreamAttributeTarget->audio_format == AUDIO_FORMAT_PCM_8_BIT ? 1 :
                                          (pStreamAttributeTarget->audio_format == AUDIO_FORMAT_PCM_16_BIT ? 2 :
                                           (pStreamAttributeTarget->audio_format == AUDIO_FORMAT_PCM_32_BIT ? 4 :
                                            (pStreamAttributeTarget->audio_format == AUDIO_FORMAT_PCM_8_24_BIT ? 4 :
                                             2))));
        const uint8_t size_per_frame = pStreamAttributeTarget->num_channels * size_per_channel;

        latency = (pStreamAttributeTarget->buffer_size * 1000) / (pStreamAttributeTarget->sample_rate * size_per_frame);
    }

    ALOGV("%s(), return %d", __FUNCTION__, latency);
    return latency;
}
```
