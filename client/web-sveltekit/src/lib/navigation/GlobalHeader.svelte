<script context="module" lang="ts">
    import { type Writable, writable, readonly } from 'svelte/store'

    export enum NavigationMode {
        PlainNavigation = 'global',
        WithCustomContent = 'with-custom-content',
    }

    const extensionElement: Writable<HTMLElement | null> = writable(null)

    export const navigationExtensionElement = readonly(extensionElement)
    export const navigationModeStore = writable<NavigationMode | `${NavigationMode}`>(NavigationMode.PlainNavigation)
</script>

<script lang="ts">
    import { browser } from '$app/environment'
    import { page } from '$app/stores'
    import { onClickOutside } from '$lib/dom'
    import Icon from '$lib/Icon.svelte'
    import { mark } from '$lib/images'
    import MainNavigationLink from '$lib/navigation/MainNavigationLink.svelte'
    import Popover from '$lib/Popover.svelte'
    import SourcegraphLogo from '$lib/SourcegraphLogo.svelte'
    import { isViewportMediumDown } from '$lib/stores'
    import { Badge, Button } from '$lib/wildcard'

    import { GlobalNavigation_User } from './GlobalNavigation.gql'
    import { type NavigationEntry, type NavigationMenu, isNavigationMenu, isCurrent } from './mainNavigation'
    import UserMenu from './UserMenu.svelte'

    export let authenticatedUser: GlobalNavigation_User | null | undefined
    export let handleOptOut: (() => Promise<void>) | undefined
    export let entries: (NavigationEntry | NavigationMenu)[]

    const isDevOrS2 =
        (browser && window.location.hostname === 'localhost') ||
        window.location.hostname === 'sourcegraph.sourcegraph.com'

    let sidebarNavigationOpen: boolean = false
    let closeMenuTimer: number = 0
    let openedMenu: string = ''

    $: withCustomContent = $navigationModeStore === NavigationMode.WithCustomContent
    $: sidebarMode = withCustomContent || $isViewportMediumDown

    function openMenu(menu: string) {
        openedMenu = menu
        clearTimeout(closeMenuTimer)
    }
    function closeMenu() {
        // We use a delay to close the menu to make it easier to navigate (back) to it
        closeMenuTimer = window.setTimeout(() => {
            openedMenu = ''
        }, 500)
    }
</script>

<header class="root" data-global-header class:withCustomContent class:sidebarMode>
    <div class="logo">
        <button class="menu-button" on:click={() => (sidebarNavigationOpen = true)}>
            <Icon icon={ILucideMenu} aria-label="Navigation menu" />
        </button>

        <a href="/search">
            <img src={mark} alt="Sourcegraph" width="25" height="25" />
        </a>
    </div>

    <nav aria-label="Main" class:as-sidebar={sidebarMode} class:open={sidebarNavigationOpen}>
        <!-- Additional wrapper needed to handle sidebar navigation mode -->
        <div
            class="content"
            use:onClickOutside={{ enabled: sidebarNavigationOpen }}
            on:click-outside={() => (sidebarNavigationOpen = false)}
        >
            <div class="sidebar-navigation-header">
                <button class="close-button" on:click={() => (sidebarNavigationOpen = false)}>
                    <Icon icon={ILucideX} aria-label="Close sidebar navigation" />
                </button>

                <a href="/search" class="logo-link">
                    <SourcegraphLogo width="9.1rem" />
                </a>
            </div>
            <ul class="top-navigation">
                {#each entries as entry (entry.label)}
                    {@const open = openedMenu === entry.label}
                    <li class:open on:mouseenter={() => openMenu(entry.label)} on:mouseleave={closeMenu}>
                        {#if isNavigationMenu(entry)}
                            <span>
                                <MainNavigationLink {entry} />
                                <Button
                                    variant="icon"
                                    on:click={() => (openedMenu = open ? '' : entry.label)}
                                    aria-label="{open ? 'Close' : 'Open'} '{entry.label}' submenu"
                                    aria-expanded={open}
                                >
                                    <Icon icon={ILucideChevronDown} inline aria-hidden />
                                </Button>
                            </span>

                            <ul class="sub-navigation">
                                {#each entry.children as subEntry (subEntry.label)}
                                    <li>
                                        <MainNavigationLink
                                            entry={subEntry}
                                            aria-current={isCurrent(subEntry, $page) ? 'page' : 'false'}
                                        />
                                    </li>
                                {/each}
                            </ul>
                        {:else}
                            <span>
                                <MainNavigationLink {entry} aria-current={isCurrent(entry, $page) ? 'page' : 'false'} />
                            </span>
                        {/if}
                    </li>
                {/each}
            </ul>
        </div>
    </nav>

    <div class="global-portal" bind:this={$extensionElement} />

    <Popover let:registerTrigger showOnHover hoverDelay={100} hoverCloseDelay={50}>
        <span class="web-next-badge" use:registerTrigger>
            <Badge variant="warning">Experimental</Badge>
        </span>
        <div slot="content" class="web-next-content">
            <h3>Experimental web app</h3>
            <p>
                You are using an experimental version of the Sourcegraph web app. This version is under active
                development and may contain bugs or incomplete features.
            </p>
            {#if isDevOrS2}
                <p>
                    If you encounter any issues, please report them in our <a
                        href="https://sourcegraph.slack.com/archives/C05MHAP318B">Slack channel</a
                    >.
                </p>
            {/if}
            {#if handleOptOut}
                Or you can <button role="link" class="opt-out" on:click={handleOptOut}>opt out</button> of the Sveltekit
                experiment.
            {/if}
        </div>
    </Popover>
    <div>
        {#if authenticatedUser}
            <UserMenu user={authenticatedUser} />
        {:else}
            <Button variant="secondary" outline>
                <svelte:fragment slot="custom" let:buttonClass>
                    <a class={buttonClass} href="/sign-in">Sign in</a>
                </svelte:fragment>
            </Button>
        {/if}
    </div>
</header>

<style lang="scss">
    .root {
        display: flex;
        align-items: center;
        gap: 0.75rem;
        padding: 0.5rem;
        border-bottom: 1px solid var(--border-color);
        background-color: var(--color-bg-1);

        position: relative;
        height: 50px;
        min-width: 0;
    }

    .logo {
        display: flex;
        align-items: center;
        margin-left: 0.5rem;
        gap: 0.5rem;

        .sidebarMode & {
            margin-left: 0;
        }

        img:hover {
            @keyframes spin {
                50% {
                    transform: rotate(180deg) scale(1.2);
                }
                100% {
                    transform: rotate(180deg) scale(1);
                }
            }

            @media (prefers-reduced-motion: no-preference) {
                animation: spin 0.5s ease-in-out 1;
            }
        }
    }

    .global-portal {
        display: none;
        flex: 1;
        align-self: stretch;
        min-width: 0;

        .withCustomContent & {
            display: flex;
        }
    }

    nav {
        ul {
            list-style: none;
            padding: 0;
            margin: 0;
        }
    }

    // Horizontal navigation style, only used on search home page
    nav:not(.as-sidebar) {
        flex: 1;
        min-width: 0;
        display: flex;
        align-self: stretch;

        .content {
            display: flex;
            align-self: stretch;
            flex: 1;
            min-width: 0;
        }

        .sidebar-navigation-header {
            display: none;
        }

        .top-navigation {
            --icon-color: var(--header-icon-color);

            display: flex;
            gap: 1rem;
            padding: 0;
            margin: -0.5rem 0 -0.5rem 0;
            position: relative;
            justify-content: center;
            background-size: contain;

            > li {
                position: relative;
                white-space: nowrap;
                border-color: transparent;

                &.open,
                &:hover {
                    border-color: var(--border-color-2);
                }

                &.open .sub-navigation {
                    display: block;
                }

                > span {
                    display: flex;
                    align-items: center;
                    gap: 0.25rem;

                    height: 100%;
                    border-bottom: 2px solid;
                    border-color: inherit;

                    :global(a) {
                        align-self: stretch;
                    }
                }

                &:has(a[aria-current='page']) {
                    border-color: var(--brand-secondary);
                }
            }
        }

        .sub-navigation {
            display: none;
            position: absolute;
            left: 0;
            right: 0;
            top: calc(100% + 3px);

            min-width: 10rem;
            background-clip: padding-box;
            background-color: var(--dropdown-bg);
            border: 1px solid var(--dropdown-border-color);
            border-radius: var(--popover-border-radius);
            color: var(--body-color);
            box-shadow: var(--dropdown-shadow);
            padding: 0.25rem 0;
            // This seems necessary to make the dropdown render above other elements
            // and keep it open when moving the mouse into it.
            z-index: 2;

            > li {
                cursor: pointer;
                display: block;
                width: 100%;
                padding: var(--dropdown-item-padding);
                white-space: nowrap;
                color: var(--dropdown-link-hover-color);

                &:hover,
                &:focus {
                    background-color: var(--dropdown-link-hover-bg);
                    color: var(--dropdown-link-hover-color);
                    text-decoration: none;
                }
            }
        }
    }

    // Sidebar navigation style
    nav.as-sidebar {
        display: none;
        top: 0;
        left: 0;
        bottom: 0;
        right: 0;
        position: fixed;
        // Fixed overlay color TODO: find a better design token for it
        background-color: rgba(172, 182, 192, 0.2);
        // Ensures that the sidebar navigation is all other elements
        z-index: 2;

        .content {
            width: 18.75rem;
            background-color: var(--color-bg-1);
            display: flex;
            flex-direction: column;
            overflow: hidden;
            min-height: 0;
            height: 100%;
        }

        &.open {
            display: block;
        }

        .sidebar-navigation-header {
            display: flex;
            gap: 0.5rem;
            align-items: center;
            padding: 0.5rem;
            // Original menu navigation has 50px - 1px bottom border
            // To ensure that there are no jumps between closed/open states
            // we set height here to repeat menu and icon buttons positions.
            min-height: 49px;
            background-color: var(--color-bg-1);

            .close-button {
                border: none;
                padding: 0.35rem 0.35rem;
                border-radius: var(--border-radius);
                display: flex;
                align-items: center;
                background-color: transparent;

                &:hover {
                    background-color: var(--secondary-2);
                }

                --icon-size: 1rem;
            }
        }

        .top-navigation {
            overflow-y: auto;
            max-width: 100vw;

            display: flex;
            flex-direction: column;
            width: 18.75rem;
            border: none;
            padding: 0;
            margin: 0;
            background-color: var(--color-bg-1);

            :global(button) {
                display: none;
            }

            :global(a) {
                display: flex;
                flex-wrap: wrap;
                align-items: center;
                gap: 0.25rem;
                padding: 0.375rem 0.75rem;
                font-size: 1rem;

                &:hover {
                    background-color: var(--secondary-2);
                }
            }
        }

        .sub-navigation {
            :global(a) {
                padding-left: 3.7rem;
            }
            :global(a[aria-current='page']) {
                background-color: var(--secondary-2);
            }
        }
    }

    // Opt out experiment badge and tooltip styles
    .opt-out {
        all: unset;
        cursor: pointer;
        color: var(--link-color);
        text-decoration: underline;
    }

    .web-next-badge {
        cursor: pointer;
        padding: 0.25rem;
        margin-left: auto;
    }

    .web-next-content {
        padding: 1rem;
        width: 20rem;

        p:last-child {
            margin-bottom: 0;
        }
    }

    // Custom menu with sidebar navigation controls styles
    .menu-button {
        display: none;
        padding: 0.35rem;
        align-items: center;
        border: none;
        background-color: transparent;
        border-radius: var(--border-radius);

        .sidebarMode & {
            display: flex;
        }

        &:hover {
            background-color: var(--secondary-2);
        }

        --icon-size: 1rem;
    }
</style>
