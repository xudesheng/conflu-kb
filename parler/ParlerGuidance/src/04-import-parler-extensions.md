# Import Parler extensions and the sample mashup

## Goal

Install the two shipped **extensions**, then import the sample **mashup**:

1. **`parler-agent`** — Java ThingWorx extension (contains **`AIAgent`**, **`ParlerGateway`**, and agent implementation).
2. **`parler-ui-widget`** — widget bundle that hosts **`<parler-ui>`**.
3. **`ParlerAgentBasic.xml`** — sample mashup that uses the `parler-ui-widget`.

Please ask David Kessler to obtain the latest extensions and XML files. In the workshop, treat these as **instructor-provided artifacts**; do not expect them to be present in the training repository.

## Build artifacts (maintainers only)

These paths are useful when you maintain the **parler** monorepo. Most workshop students receive the finished ZIP/XML files instead.

- Agent ZIP: `parler-agent/build/parler-agent.zip` after **`./build-extension.sh`** (or equivalent Gradle assemble).
- Widget: follow **`./build-widget.sh`** and **`parler-ui-widget/README.md`** so Composer receives the current **`parler-ui`** bundle.

## Verification (screenshots)

After you import the two extensions, you should be able to find them on the `Installed Extensions` page.

<img src="./__images__//image-20260530015642854.png" alt="image-20260530015642854" style="zoom:50%;" />



Once you have imported the `ParlerAgentBasic.xml` file, you should be able to see the `Parler` mashup.

<img src="./__images__//image-20260530171358622.png" alt="image-20260530171358622" style="zoom:50%;" />
