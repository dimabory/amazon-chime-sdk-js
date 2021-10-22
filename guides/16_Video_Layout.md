# Managing Video Quality for Different Video Layouts

## Overview

The Amazon Chime SDK for JavaScript enables builders to create applications with custom and dynamic video layouts. Depending on the number of video tiles displayed on the screen, their relative display size and priority, different video upstream strategies and tile management methods are applied to provide the optimal video experience for end users.

The first section introduces the basic concepts, including video lifecycle management APIs and the uplink and downlink policies. The second section discusses several common use cases and how to implement them using the [Amazon Chime SDK for JavaScript](https://github.com/aws/amazon-chime-sdk-js) (“JavaScript SDK”) and the [Amazon Chime SDK React Components Library](https://github.com/aws/amazon-chime-sdk-component-library-react) (“React Components Library”).

## Introduction

### Video Lifecycle Management APIs

Let us consider a scenario where a meeting has three attendees: attendee 0, attendee 1, and attendee 2. They are all sharing their local camera stream in a meeting, and attendee 2 is additionally sharing their screen via content share. The diagram below illustrates how all the video streams are received and managed from the perspective of attendee 0.

![video lifecycle diagram](media://video_lifecycle.png)

Typically, there are four steps involved in the lifecycle of a remote video with respect to a single receiving attendee:

1. As remote attendees change their video stream, an up-to-date list of video sources (ignoring the receiving attendees’ own local video) is provided via the `remoteVideoSourcesDidChange` callback.
2. The receiving client will select the remote video sources (and possibly which simulcast layer) that they wish to receive, which will initiate a negotiation (or “subscription”) with the backend servers to start forwarding those videos, depending on a client’s downlink policy (explained in a later section).

   - Note that some downlink policies may expose APIs to allow dynamic control over this decision.

3. For each subscribed or unsubscribed video, the receiving attendee will receive a `videoTileDidUpdate` or `videoTileWasRemoved` callback.
4. To display the video, the receiving attendee can bind or unbind a video tile to a  `<video>` element in their canvas via the [`bindVideoElement`](https://aws.github.io/amazon-chime-sdk-js/interfaces/audiovideofacade.html#bindvideoelement) API.

### Video Uplink and Downlink Policies

Uplink and downlink policies help builders configure the video strategies for sending and receiving video streams in a meeting. The Amazon Chime SDK for JavaScript provides different policies, each with its own configuration and dynamic controls. Builders can choose between the provided policies or extend and create their own, depending on the requirements.

#### Uplink Policy

The video encoding parameters are determined by the uplink policy, which takes into consideration the number of video publishers on the call. If necessary, it may suggest encoding multiple “simulcast” layers (with different bitrates and resolutions). The JavaScript SDK currently provides two uplink policies:

**NScaleVideoUplinkBandwidthPolicy (Default):** In this policy, an attendee encodes and sends one video layer upstream and scales the resolution and bitrate based on the number of video publishers. This is to ensure the receiving attendees who are subscribed to these videos do not experience excessive network or compute load, which may degrade the meeting quality.

**SimulcastUplinkPolicy:** Each attendee may encode and send up to two layers of video with different bitrates and resolutions under this policy. It enables the recipient to switch between the two video layers to adapt to changing network conditions or different layouts (e.g., a featured video may desire a higher resolution than a gallery view video).

- Note: the `SimulcastUplinkPolicy` is currently only supported in Chrome 76 and above versions.

#### Downlink Policy

The downlink policy controls which remote video sources are received downstream based on the network condition, local resources (e.g., CPU), and other factors such as layout changes in the application. The JavaScript SDK currently provides two downlink policies:

**AllHighestVideoBandwidthPolicy (Default):** This policy subscribes to the highest quality video layer of all publishers. Under constrained network, some or all videos may freeze as packets are lost on the network, and they will recover automatically once network congestion is reduced.

**VideoPriorityBasedPolicy:** This policy allows builders to choose and configure which video sources to subscribe to, their relative priorities and preferences such as, `targetDisplaySize` . Under constrained network, the policy will automatically pause video tiles based on the priority of the video sources until recovery. If used with the `SimulcastUplinkPolicy`, the policy may temporarily degrade to lower resolutions. For videos of the same priority, the policy first tries to allocate bandwidth to ensure as many videos can be displayed as possible before upgrading to higher quality if the bandwidth allows.

## Recommendations for Common Layouts

### Case 1: Small Grid Layout (up to 4 Videos) with Fixed Tile Size

We recommend the default configuration of the `NScaleVideoUplinkBandwidthPolicy`  and  `AllHighestVideoBandwidthPolicy` when displaying a small number of video tiles of a fixed size, so the recipient is less likely to get overwhelmed. Video freezes can occur due to network congestion and recover automatically.

#### JavaScript SDK Implementation

Refer to code sample in [Use case 16](https://github.com/aws/amazon-chime-sdk-js#video).

#### React Components Library Implementation

The different callbacks are already handled by the React Components Library. If you don’t have the content sharing functionality in your application, you can use the [`VideoTileGrid`](https://aws.github.io/amazon-chime-sdk-component-library-react/?path=/story/sdk-components-videotilegrid--page) component with the `standard` layout from the React Components Library.

```jsx
import React from 'react';
import {
  MeetingProvider,
  VideoTileGrid
} from 'amazon-chime-sdk-component-library-react';

const App = () => (
  <MeetingProvider>
    <VideoTileGrid layout="standard"/>
  </MeetingProvider>
);
```

Otherwise, you need to build a custom video grid using the components and hooks from the React Components Library.

```jsx
import {
  LocalVideo,
  RemoteVideos,
  useLocalVideo,
  useRemoteVideoTileState,
  VideoGrid,
} from 'amazon-chime-sdk-component-library-react';
import React from 'react';

const fluidStyles = `
  height: 100%;
  width: 100%;
`;

const staticStyles = `
  display: flex;
  position: absolute;
  bottom: 1rem;
  right: 1rem;
  width: 20vw;
  max-height: 30vh;
  height: auto;

  video {
    position: static;
  }
`;

export const GalleryVideoTileGrid: React.FC = () => {
  const { tiles } = useRemoteVideoTileState();
  const { isVideoEnabled } = useLocalVideo();
  const gridSize = isVideoEnabled ? tiles.length + 1 : tiles.length;

  return (
    <VideoGrid size={gridSize} layout='standard'>
      <RemoteVideos />
      <LocalVideo
        nameplate="Me"
        css={gridSize > 1 ? fluidStyles : staticStyles}
      />
    </VideoGrid>
  );
};

export default GalleryVideoTileGrid;
```

### Case 2: Gallery Layout (up to 25 Videos) with Fixed Tile Size

While the JavaScript SDK supports viewing up to [25 video tiles](https://docs.aws.amazon.com/chime/latest/dg/meetings-sdk.html#mtg-limits) at this time, as the number of video publishers increases, the bandwidth and CPU consumption of receiving clients may not be sufficient to display all videos.

![gallery layout example](media://gallery_layout_example.png)

The default setting of `NScaleVideoUplinkBandwidthPolicy` and `AllHighestVideoBandwidthPolicy` can result in video freezes as the number of videos goes up and more videos have to be decoded. Video packet loss and network delays are a result of high CPU consumption and can impact all streams equally. This may not be acceptable if the application wishes to preserve certain videos over others, and it can be mitigated by the application directly calling the `pauseReceivingStream` API to pause certain videos.

The `VideoPriorityBasedPolicy` is recommended as the downlink policy if you wish to proactively pause (video will not be displayed) some videos to save the bandwidth in order to preserve others. This gives the application builder more control and makes the impact on the video experience resulting from network constraints more predicable. The `VideoPriorityBasedPolicy` triggers callbacks if video tiles are automatically paused or unpaused so that you can properly notify the users and potentially replace the empty video tile with something like avatars to reduce the visual impact.

#### JavaScript SDK Implementation

Refer to code sample in [`User Guide for PriorityBased Downlink Policy`](https://aws.github.io/amazon-chime-sdk-js/modules/prioritybased_downlink_policy.html#builder-code-sample) and [use case 24](https://github.com/aws/amazon-chime-sdk-js#attendees).

#### React Components Library Implementation

If you don’t have content sharing functionality in your application, you can use the [`VideoTileGrid`](https://aws.github.io/amazon-chime-sdk-component-library-react/?path=/story/sdk-components-videotilegrid--page) component with the `standard` layout from React Components Library. Otherwise, you need to build a custom video grid using the components and hooks from the React Components Library. Refer to the implementation part under case 1 for details.

Create a `meetingConfig` object to store the configuration of `MeetingProvider`. This configuration is passed down to the `MeetingManager` and is used to initialize the `MeetingSession` when joining a meeting.

```jsx
import {
  ConsoleLogger,
  LogLevel,
  VideoPriorityBasedPolicy,
} from 'amazon-chime-sdk-js';

// Initialize `VideoPriorityBasedPolicy` and export it to use elsewhere.
const logger = new ConsoleLogger('SDK', LogLevel.INFO);
export const priorityBasedPolicy = new VideoPriorityBasedPolicy(logger);

const meetingConfig = {
  logger,
  videoDownlinkBandwidthPolicy: priorityBasedPolicy,
};

export default meetingConfig;
```

Setup the `MeetingProvider` using the `meetingConfig`.

```jsx
import React from 'react';
import { MeetingProvider } from 'amazon-chime-sdk-component-library-react';
import meetingConfig from './meetingConfig';

const App = () => (
  <MeetingProvider {...meetingConfig}>
    <MyApp />
  </MeetingProvider>
);
```

Create a [`Provider`](https://zh-hans.reactjs.org/docs/context.html#contextprovider) to use `priorityBasedDownlinkPolicy`.

1. Import the previous `VideoPriorityBasedPolicy` instance in your provider.
2. Create an observer and subscribe to the `tileWillBePausedByDownlinkPolicy` and `tileWillBeUnpausedByDownlinkPolicy` callbacks, so you can update the UI to notify the user that the related video tile has been paused or unpaused.
3. Create an observer and subscribe to the `remoteVideoSourcesDidChange` callback, so you can track the states of remote video sources for future subscriptions. You need a variable like `roster: any = {}` to keep track of which attendees in the meeting are publishing videos. Check [use case 24](https://github.com/aws/amazon-chime-sdk-js#attendees) for more details.

   - Note: you may want to keep the content sharing in your `roster`, depending on your needs.

```jsx
import { useAudioVideo } from 'amazon-chime-sdk-component-library-react';
import { priorityBasedPolicy } from './meetingConfig';

const GridStateProvider: React.FC = ({ children }) => {
  const audioVideo = useAudioVideo();

  // Subscribe to `tileWillBePausedByDownlinkPolicy` and
  // `tileWillBeUnpausedByDownlinkPolicy` callbacks.
  useEffect(() => {
    if (!priorityBasedPolicy || !audioVideo) {
      return;
    }

    const observer: VideoDownlinkObserver = {
      tileWillBePausedByDownlinkPolicy: (tileId: number): void => {
        const attendeeId = audioVideo.getVideoTile(tileId)?.state().boundAttendeeId;
        if (attendeeId) {
          // Update the UI to notify user the pause of
          // video tile according to below property.
          roster[attendeeId].paused = true;
        }
      },
      tileWillBeUnpausedByDownlinkPolicy: (tileId: number): void => {
        const attendeeId = audioVideo.getVideoTile(tileId)?.state().boundAttendeeId;
        if (attendeeId) {
          // Update the UI to notify user the unpause 
          // of video tile according to below property.
          roster[attendeeId].paused = false;
        }
      },
    };

    priorityBasedPolicy.addObserver(observer);

    return (): void => priorityBasedPolicy.removeObserver(observer);
  }, [audioVideo]);

  // Subscribe to `remoteVideoSourcesDidChange` callback.
  useEffect(() => {
    if (!audioVideo) {
      return;
    }

    const observer: AudioVideoObserver = {
      remoteVideoSourcesDidChange: (videoSources: VideoSource[]): void => {
        // Update your `roster` object to track the video sources state.
        for (const attendeeId in roster) {
          roster[attendeeId].hasVideo = false;
        }
        for (const source of videoSources) {
          const attendeeId = source.attendee.attendeeId;
          roster[attendeeId].hasVideo = true;
        }
      },
    };

    audioVideo.addObserver(observer);

    return (): void => audioVideo.removeObserver(observer);
  }, [audioVideo]);

  return (
    <GridStateContext.Provider value= { value } >
    { children }
    < /GridStateContext.Provider>
  );
};
```

Call the [`chooseRemoteVideoSources`](https://aws.github.io/amazon-chime-sdk-js/modules/prioritybased_downlink_policy.html#apis-and-usage) API to set which video sources to receive and their related preferences.

- Note: [`VideoPreference`](https://aws.github.io/amazon-chime-sdk-js/classes/videopreference.html)`(attendeeId: string, priority: number, targetSize?: TargetDisplaySize)`. The first property `attendeeId` specifies the video source. The second property `priority` is the relative priority of this attendee against others. The lower the number, the higher the priority. The third property `targetSize` is the desired maximum simulcast layers to receive. It doesn’t work in this example, since `NScaleVideoUplinkBandwidthPolicy` only publishes one layer.

```jsx
// In this example, only render up to ten video sources.

const updateVideoPreferences = () => {
  // Initialize `VideoPreferences`.
  const videoPreferences = VideoPreferences.prepare();

  let numVideos = 0;

  for (const attendeeId in roster) {
    if (roster[attendeeId].hasVideo && numVideos < 10) {
      // Subscribe to the first ten video sources
      // and set highest priority for them.
      videoPreferences.add(
        new VideoPreference(attendeeId, 1, TargetDisplaySize.High)
      );
      numVideos += 1;
    }
    // The remaining video sources are not subscribed,
    // so they do not cost bandwidth.
  }

  priorityBasedDownlinkPolicy.chooseRemoteVideoSources(videoPreferences.build());
};
```

### Case 3: Featured Layout

The featured layout is designed to focus on important video sources such as an active speaker or content share. The featured tile displays a video source in a large area while other video sources (up to [25 video tiles](https://docs.aws.amazon.com/chime/latest/dg/meetings-sdk.html#mtg-limits)) are displayed as smaller tiles.

![featured layout example](media://featured_layout_example.png)

For layouts where all users feature the same content source or active speaker, the `NScaleVideoUplinkBandwidthPolicy` and `VideoPriorityBasedPolicy` are recommended.  The `NScaleVideoUplinkBandwidthPolicy` automatically prioritizes the content share and boosts the resolution of the active speaker video. The `VideoPriorityBasedPolicy` ensures the bandwidth is efficiently used to display only the video in the featured tile.

- Note: ensure that you’re listening to changes in active speakers and passing them to the `VideoPriorityBasedPolicy`.

#### JavaScript SDK Implementation

Setup the `VideoPriorityBasedPolicy`, as described in [case 2](#javascript-sdk-implementation-1).

Listen to the changes in active speakers and track their states.

```js
import { DefaultActiveSpeakerPolicy } from 'amazon-chime-sdk-js';

const activeSpeakerCallback = (attendeeIds: string[]): void => {
  // Reset `active` property
  for (const attendeeId in roster) {
    roster[attendeeId].active = false;
  }

  // Set `active` to true for the most active speaker.
  // The `attendeeIds` returned by the `activeSpeakerCallback` 
  // are sorted by the most active
  for (const attendeeId of attendeeIds) {
    if (roster[attendeeId]) {
      roster[attendeeId].active = true;
      break;
    }
  }

  // Update video preferences whenever there are changes of active speakers.
  updateVideoPreferences();
};

meetingSession.audioVideo.subscribeToActiveSpeakerDetector(
  new DefaultActiveSpeakerPolicy(),
  activeSpeakerCallback
);
```

Update video preferences whenever there are changes to active speakers.

```js
// Video source priority in this example:
// Content share > active speakers > others

const updateVideoPreferences = () => {
  // Initialize `VideoPreferences`.
  const videoPreferences = VideoPreferences.prepare();

  for (const attendeeId in roster) {
    if (roster[attendeeId].hasVideo) {
      if (new DefaultModality(attendeeId).hasModality(DefaultModality.MODALITY_CONTENT)) {
        // Prioritize content share.
        videoPreferences.add(
          new VideoPreference(attendeeId, 1, TargetDisplaySize.High)
        );
      } else if (roster[attendeeId].active) {
        // Prioritize active speakers.
        videoPreferences.add(
          new VideoPreference(attendeeId, 2, TargetDisplaySize.High)
        );
      } else {
        // Handle the rest of video sources.
        videoPreferences.add(
          new VideoPreference(attendeeId, 3, TargetDisplaySize.Low)
        );
      }
    }
  }

  priorityBasedDownlinkPolicy.chooseRemoteVideoSources(videoPreferences.build());
};
```

#### React Components Library Implementation

You can use the [`VideoTileGrid`](https://aws.github.io/amazon-chime-sdk-component-library-react/?path=/story/sdk-components-videotilegrid--page) component from the React Components Library. The `featured` layout is used by default.

- Note: the `featured` layout `VideoTileGrid` component displays either the content sharing video or the active speaker video in the featured tile. If both are present, the content sharing video is displayed. If neither is present, there is no featured tile and all video tiles are of fixed and equal size, like in the gallery layout.

```jsx
import React from 'react';
import {
  MeetingProvider,
  VideoTileGrid
} from 'amazon-chime-sdk-component-library-react';

const App = () => (
  <MeetingProvider>
    <VideoTileGrid />
  </MeetingProvider>
);
```

Then create a provider to handle the `VideoPriorityBasedPolicy` and video preferences update logic.

Setup `VideoPriorityBasedPolicy`, as described in the [case 2](#react-components-library-implementation-1).

Listen to the changes in active speakers and track their states.

- The React Components Library provides the [`useActiveSpeakersState`](https://aws.github.io/amazon-chime-sdk-component-library-react/?path=/story/sdk-hooks-useactivespeakersstate--page) hook that returns a list of attendee IDs of the active speaker. Use this hook directly.

```jsx
import { useAudioVideo, useActiveSpeakersState } from 'amazon-chime-sdk-component-library-react';
import { priorityBasedPolicy } from './meetingConfig';

const GridStateProvider: React.FC = ({ children }) => {
  const audioVideo = useAudioVideo();
  const activeSpeakers = useActiveSpeakersState();

  // Set up observers and subscribe to callbacks.
  // ...
  
  useEffect(() => {
    // Reset `active` property
    for (const attendeeId in roster) {
      roster[attendeeId].active = false;
    }

    // Set `active` to true for the most active speaker.
    // The `activeSpeakers` returned by the `useActiveSpeakersState`
    // hook are sorted by the most active.
    for (const attendeeId of activeSpeakers) {
      if (roster[attendeeId]) {
        roster[attendeeId].active = true;
        break;
      }
    }

    // Update video preferences whenever there are changes of active speakers.
    updateVideoPreferences();
  }, [activeSpeakers]);

  return (
    <GridStateContext.Provider value= { value } >
    { children }
    < /GridStateContext.Provider>
  );
};
```

Update video preferences whenever there are changes to active speakers, as described in the above [JavaScript SDK Implementation](#javascript-sdk-implementation-3).

For layouts where different users feature different videos, the `SimulcastUplinkPolicy` and `VideoPriorityBasedPolicy` are recommended. The `SimulcastUplinkPolicy` sends multiple resolutions of the same video to the Chime servers. The `TargetDisplaySize` property can be specified according to the layout for each user in the `VideoPriorityBasedPolicy`, which allows for featured video to be rendered in high quality and the other videos are efficiently rendered according to display size.

#### JavaScript SDK Implementation

Setup `VideoPriorityBasedPolicy`, as described in [case 2](#javascript-sdk-implementation-1).

Enable the below feature flags in the [`MeetingSessionConfiguration`](https://aws.github.io/amazon-chime-sdk-js/classes/meetingsessionconfiguration.html) to use `SimulcastUplinkPolicy`. Refer to the code sample in the [User Guide for Simulcast](https://aws.github.io/amazon-chime-sdk-js/modules/simulcast.html#creating-a-simulcast-enabled-meeting) for details.

```js
configuration.enableUnifiedPlanForChromiumBasedBrowsers = true;
configuration.enableSimulcastForUnifiedPlanChromiumBasedBrowser = true;
```

Update the video preferences whenever users feature or unfeature a video.

```js
const featureVideo = (attendeeId: string) => {
  roster[attendeeId].featured = true;
  updateVideoPreferences();
};

const unfeatureVideo = (attendeeId: string) => {
  roster[attendeeId].featured = false;
  updateVideoPreferences();
};

const updateVideoPreferences = () => {
  // Initialize `VideoPreferences`.
  const videoPreferences = VideoPreferences.prepare();

  for (const attendeeId in roster) {
    if (roster[attendeeId].hasVideo) {
      if (roster[attendeeId].featured) {
        // Prioritize featured video sources.
        videoPreferences.add(
          new VideoPreference(attendeeId, 1, TargetDisplaySize.High)
        );
      } else {
        // Handle the rest of video sources.
        videoPreferences.add(
          new VideoPreference(attendeeId, 2, TargetDisplaySize.Low)
        );
      }
    }
  }

  priorityBasedDownlinkPolicy.chooseRemoteVideoSources(videoPreferences.build());
};
```

#### React Components Library Implementation

Setup `VideoPriorityBasedPolicy`, as described in [case 2](#react-components-library-implementation-1).

Update the `meetingConfig` object to use `SimulcastUplinkPolicy`.

```jsx
import {
  ConsoleLogger,
  LogLevel,
  VideoPriorityBasedPolicy,
} from 'amazon-chime-sdk-js';

const logger = new ConsoleLogger('SDK', LogLevel.INFO);
export const priorityBasedPolicy = new VideoPriorityBasedPolicy(logger);

const meetingConfig = {
  // Add this line to use `SimulcastUplinkPolicy`.
  simulcastEnabled: true,
  logger,
  videoDownlinkBandwidthPolicy: priorityBasedPolicy,
};

export default meetingConfig;
```

Setup the `MeetingProvider` using the `meetingConfig`.

```jsx
import React from 'react';
import { MeetingProvider } from 'amazon-chime-sdk-component-library-react';
import meetingConfig from './meetingConfig';

const App = () => (
  <MeetingProvider {...meetingConfig}>
    <MyApp />
  </MeetingProvider>
);
```

Update the video preferences whenever users feature a video, as described in the above [JavaScript SDK Implementation](#javascript-sdk-implementation-3).
