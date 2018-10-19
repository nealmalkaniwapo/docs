# Header Bidding

This doc will walk you through all the requirements and implementation steps needed to implement Header Bidding with ArcAds, PreBid and/or Amazon A9. ArcAds supports integrations with `PreBid` and `Amazon A9`.

## Requirements Before Coding Away:
1. Get your clients DFP ID
2. Determine what vendors you need to integrate with (OpenX, AppNexus, A9, etc.)
3. Determine which ones you will be integrating through PreBid.
4. For each of the vendors, you need the vendor specific `IDs` for *each* ad slot. It all depends on how your client has configured header bidding with each vendor (there could be 100+ `IDs` for each vendor or just a couple `IDs`). To figure out what values you need for each bidder go to [PreBid Bidders](http://prebid.org/dev-docs/bidders.html) and select the vendor. The params you need are all the fields listed as required.
5. You need the `pos` key value pair to determine ad position targetting (optional)
5. Get all the size mapping from your client for each ad slot/type

### Examples of client provided data:

The 2 code snippets below are examples of all the data you need from your client before you can start implementing Header Bidding.

This code shows you a snippet of an 'Ad Catalog' for a client. It contains the `ad name`, `DFP Position`, `pos` key/value pair, and `sizes`. 

#### Ad Catalog
```
[AD_BILLBOARD]: {
    DFPPosition: AD_BILLBOARD,
    pos: ['billboard'],
    defaultSizes: [_970x250],
    sizesByBreakpoint: {
        tablet: [_768x250, _768x90, _728x90],
        desktop: [_970x250, _970x90, _728x90]
    },
    bidderEnabled: true
},
[AD_BILLBOARD2]: {
    DFPPosition: AD_BILLBOARD2,
    pos: ['billboard2', 'btf'],
    defaultSizes: [_970x250],
    sizesByBreakpoint: {
        mobile: [_320x100, _320x50],
        tablet: [_728x90, _600x250],
        desktop: [_970x250, _970x90, _728x90, _600x250]
    },
    bidderEnabled: true
}
```

The below code snippet shows you the values we need to integrate the above 2 code slots with OpenX. In this set up, the client has set up a very granular configuration with OpenX because for every ad slot/placement, they have a deeper level of targetting based on the specific category of the page the user is on:
```
[AD_BILLBOARD]: {
    [HOMEPAGE]: 123455318,
    [TODAYSPAPER]: 123455321,
    [METRO]: 123455324,
    [BUSINESS]: 123455343,
    [OPINION]: 123455354,
    [IDEAS]: 123455365,
    [POLITICS]: 123455376,
    [LIFESTYLE]: 123455388,
    [ARTS]: 123455399,
    [SPORTS]: 123455410
},
[AD_BILLBOARD2]: {
    [HOMEPAGE]: 67895318,
    [TODAYSPAPER]: 67895321,
    [METRO]: 67895324,
    [BUSINESS]: 67895343,
    [OPINION]: 67895354,
    [IDEAS]: 67895365,
    [POLITICS]: 67895376,
    [LIFESTYLE]: 67895388,
    [ARTS]: 67895399,
    [SPORTS]: 67895410
}
```

Similar to the example above, you will need something similar for each vendor you need to integrate with (except for Amazon A9 - they are a lot easier to set up)

## Implementation Steps:
1. Download and include the PreBid script from [here](http://prebid.org/download.html)
2. Create the `Ad Catalog` object (example snippet above). This will contain all the information you need to pass to ArcAds that is vendor agnostic.
3. Create a specific catalog object for each vendor - this needs to be able to map to the main `Ad Catalog` to easily reference back and forth.
4. Now you are ready to work with ArcAds...

First, create an instance of ArcAds as a component and fill in all the client specific information like `dfp.id`, `bidding.amazon.id` and `prebid.sizeConfig.mediaQuery`:

```
import { PureComponent } from 'react';
import { ArcAds as WArcAds } from 'arcads';

export default class ArcAdsInstance extends PureComponent {

  static myInstance;

  static getInstance() {
    if (ArcAdsInstance.myInstance == null) {
      ArcAdsInstance.myInstance = new ArcAdsInstance();
    }
    return this.myInstance;
  }

  registerAd(params, dfpPublisherID, amazonBiddingPubID, sizesByBreakpoint) {
    if (!this.arcAdsB) {
      this.arcAdsB = new WArcAds({
        dfp: {
          id: dfpPublisherID
        },
        bidding: {
          amazon: {
            enabled: true,
            id: amazonBiddingPubID
          },
          prebid: {
            enabled: true,
            timeout: 2000,
            sizeConfig: [
              {
                'mediaQuery': '(min-width: 1024px)',
                'sizesSupported': sizesByBreakpoint.desktop,
                'labels': ['desktop']
              }, 
              {
                'mediaQuery': '(min-width: 480px) and (max-width: 1023px)',
                'sizesSupported': sizesByBreakpoint.tablet,
                'labels': ['tablet']
              }, 
              {
                'mediaQuery': '(min-width: 0px)',
                'sizesSupported': sizesByBreakpoint.mobile,
                'labels': ['phone']
              }
            ]
          }
        }
      });
    }
    this.arcAdsB.registerAd(params);
  }
}
```

Next, you need to call this component will all the params you set up in the `Ad Catalog`:

```
arcAdsInstance.registerAd({
    id: <this is the DFPPosition from the Ad Catalog>,
    slotName: this.adUnitCode,***
    dimensions = [ 
        <Pass the desktop sizemap from the item in the Ad Catalog> || [],
        <Pass the tablet sizemap from the item in the Ad Catalog> || [],
        <Pass the mobile sizemap from the item in the Ad Catalog> || [] 
    ];
    sizemap = {
        breakpoints: [ [1200, 0], [960, 0], [768, 0] ],
        refresh: true
    },
    display: <you can let the client choose from the following values: `all`, `desktop`, `mobile`>,
    targeting: { < all the key value pairs your client wants (like `pos`)> },
    bidding: {
        amazon: {
            enabled: < true or false >,
        },
        prebid: {
        enabled: < true : false >,
        bids: [{
                bidder: 'openx',
                params: {
                    delDomain: < OpenX Domain - get from client >,
                    unit: < OpenX Unit/ID >
                }
            },{
                bidder: 'appnexus',
                params: {
                    placementId: < AppNexus ID >
                }
        }]
        }
    }
    }, 
    dfpPublisherID,
    amazonBiddingPubID,
    sizesByBreakpoint
)
  ```

  Important syntax to note from the snippet above:
  1. `dimensions` needs to either be a 3d or 2d array. For example:
  * 3d Array: `[ [ [desktop dimensions 1],[desktop dimensions 2] ], [ [tablet dimensions 1],[tablet dimensions 2] ], [ [mobile dimensions 1],[mobile dimensions 2] ] ]`
  * 2d Array: `[ [ dimension 1 ], [ dimension 2 ], [ dimension 3 ] ]`

  2. If you are passing a 3 dimensional array - the number of sections needs to match up with the number of `sizemap.breakpoints`. For example the sizemap below has 3 breakpoints. So the `dimensions` also needs to have 3 sections (S1, S2, S3):
  * `sizemap.breakpoints` = `[ [1200, 0], [960, 0], [768, 0] ]`
  * `dimensions` = `[ [S1 - Contains Array(int)], [S2 - Contains Array(int)], [S3 - Contains Array(int)] ]`


## Testing
This is where it gets ...more... interesting. A lot of your testing is going to dependent on working with the vendors.

### Prebid - Testing Requesting Bids:
1. Include the query param `?pbjs_debug=true` to the end of your testing URL
2. Open the console and you should see all the PreBid debug messages roll in
3. You should see a line that reads: `Bids Requested for Auction with ID: [{...},{...}...]`. You can expand that array and confirm the request you submitted has the expected parameters.
4. You can also go into the Netowrk tab and see the ad requests go out. What to look for is different for each vendors request. For OpenX, you can search for `arj?`.

### The Tricky Part - Getting Bids

1. Ask each vendor to whitelist/greylist your sandbox/testing environent. Otherwise, your bid requests will go through but since they don't recognize where the request originated they will not respond with any bids.
2. Even after they whitelist/greylist your environment, there is still a chance you don't recieve any bids in return. But this situation is expected even when the site is live. In this case your ads should still render from Direct Sales in DFP (handled by ArcAds if configuration is correct).
3. Your last way to test is to give a testing URL to the vendor and have them validate they are able to see the request correctly through their internal tools.