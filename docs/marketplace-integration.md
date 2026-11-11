# Use AWS Marketplace images

VEW can import AMIs from your AWS Marketplace subscriptions into the image catalog. Once imported, you publish them as products the same way you would any other image — the only difference is how VEW handles licensing in the spoke account.

## Subscription ownership models

When you discover marketplace subscriptions in VEW, each one is classified into one of two models. The model determines what happens during publishing.

### VEW Managed subscription

The hub account subscribed directly to the product, and the license permits re-granting. When you publish a version that uses this image, VEW automatically creates a License Manager grant from the hub to the spoke account, accepts it, and activates it. You don't need to do anything on the spoke side.

### Externally Managed subscription

The hub received the license from outside — a vendor grant, an Organization-level distribution, or a similar arrangement. VEW cannot redistribute it automatically. Each spoke account must already have an active grant (distributed by an Organization admin) or its own independent Marketplace subscription before you publish. VEW checks for this at publish time and fails with a clear error if neither is present.

## Prerequisites

These are one-time setup steps performed by an AWS administrator in the hub account.

### Enable License Manager Organization integration

1. Go to **AWS Marketplace → Settings → AWS License Manager integration → View setting details** and choose **Create integration**. This creates the `AWSServiceRoleForMarketplaceLicenseManagement` service-linked role.
2. Go to **License Manager → Settings → Account details** and choose **Link AWS Organizations accounts**.

This enables cross-account license sharing. You can verify it worked by checking **IAM → Roles** for the `AWSServiceRoleForMarketplaceLicenseManagement` role.

### Subscribe to a Marketplace product

For VEW Managed subscriptions, the hub account needs its own purchase:

1. Go to **AWS Marketplace → Discover products** in the hub account.
2. Subscribe to an AMI product. Prefer products with hourly-only pricing — products with annual contract pricing require a payment method on the account and will fail the agreement creation if one isn't configured.
3. Wait a few minutes for the subscription to activate.

For Externally Managed subscriptions, skip this step. The licenses already exist through external distribution.

### Distribute grants to spoke accounts (Externally Managed only)

If you're working with Externally Managed subscriptions, an Organization admin must distribute grants to each spoke account that will run workbenches:

1. In the hub account, go to **License Manager → Granted licenses**, select the license, and choose **Create grant**.
2. Enter the spoke account ID.
3. In the spoke account, accept and activate the grant.

## Discover subscriptions

1. Go to **Project → Images → Import AMI → Marketplace Product**.
2. VEW lists all active Marketplace subscriptions in the hub account. Each subscription shows a badge indicating its ownership model — **VEW Managed** or **Externally Managed**.

If the list is empty, the hub account has no active Marketplace subscriptions. Subscribe to a product or ask your Organization admin to grant a license.

## Import a Marketplace AMI

1. From the import wizard, select **Marketplace Product** and pick a subscription.
2. Proceed to the image details step. Fill in the **Image Name**, **OS Version**, and at least one **Component Version**.
3. Review the summary and choose **Import**.

The image appears in the catalog with a purple **Marketplace** badge. The detail page includes a Marketplace Metadata section with subscription details.

If you've already imported a particular product, it shows as **Already Imported** in the subscription list (dimmed and non-selectable).

## Build on a Marketplace image

A recipe version can use an imported Marketplace image as its base image instead of the default operating system image, so you can add your own components on top of the vendor's AMI.

1. Import the Marketplace product into the image catalog as described above.
2. Create or update a recipe version. Under **Base image**, pick the imported image. The list shows the Marketplace images in the catalog that have the **Created** status and match the recipe's platform and architecture. Pick **Default operating system image** to build on the default image again.
3. Build the recipe version through a pipeline as usual.

Images built from the recipe version keep the Marketplace product code. VEW records the base image's product code and license details on them. When you publish a product version from such an image, VEW shares the image with the spoke account like any other built image, and then handles the license in the spoke account as described under [Publish a product version](#publish-a-product-version).

Keep in mind:

- **Only images that boot from `/dev/sda1`.** Recipes, recipe testing and the product templates map the root volume to `/dev/sda1`, so VEW rejects images with another root device. The recipe version's volume size must be at least the size of the image's root volume.
- **The base AMI is pinned per recipe version.** When you create or update a recipe version, VEW resolves the image's AMI in the region recipes are built in and keeps building on that AMI. Importing a newer version of the product retires the old image in the catalog but doesn't change existing recipe versions, and you can keep updating a recipe version that builds on the retired image. To move a recipe to the new AMI, update a recipe version or create a new one and pick the new image.
- **The Image Service account needs its own license.** Image Builder and recipe testing launch the Marketplace AMI in the Image Service account. When that account is separate from the hub, it needs its own subscription to the product or a License Manager grant for it. Like any base image, the AMI also needs the SSM Agent for Image Builder and recipe testing.

## Publish a product version

Create a product, then create a version selecting the marketplace-imported image. From here, the process is the same as any other product version — but what happens behind the scenes depends on the ownership model.

### VEW Managed license

VEW creates a License Manager grant from the hub to the spoke, accepts it in the spoke account, and waits for the launch permission to propagate. This is fully automatic. The version status progresses to **Created** once complete.

### Externally Managed license

VEW checks that the spoke account already has an active grant or its own subscription for the product. If it does, the version proceeds normally — no new grant is created.

If the spoke account has neither, the version fails with an error indicating that an Organization admin must distribute the license to the spoke account. To fix this, create and accept a grant from the hub to the spoke (see [Distribute grants to spoke accounts](#distribute-grants-to-spoke-accounts-externally-managed-only) above), then retry the version.
