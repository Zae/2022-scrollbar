<template>
    <aside
        id="outline"
        ref="outline"
        :class="{ visible: isVisible }"
    >
        <section class="nav">
            <button :class="{ visible: isTab('headings') }" @click="switchTab('headings')">HEADINGS</button>
            <button :class="{ visible: isTab('navs') }" @click="switchTab('navs')">NAVS</button>
        </section>

        <section class="tab" v-if="isTab('headings')" data-tab="headings">
            <span
                v-for="(node, key) in headings"
                :key="key"
                v-text="node && node.title"
                :title="node && node.title"
                class="node"
                :class="{ visible: node && node.visible, enabled: node }"
                @click="onClick(node && node.element)"
            />
        </section>

        <section class="tab" v-if="isTab('navs')" data-tab="nav">
            <span
                v-for="(node, key) in navs"
                :key="key"
                v-text="node && node.title"
                :title="node && node.title"
                class="node"
                :class="{ visible: node && node.visible, enabled: node }"
                @click="onClick(node && node.element)"
            />
        </section>
    </aside>
</template>

<script>
import debounce from 'lodash/debounce';
import throttle from 'lodash/throttle';

const slotHeight = 15;

export default {
    props: {
        headingsSelector: {
            type: String,
            default: 'h1, h2, h3, h4, h5, h6'
        },
        navSelector: {
            type: String,
            default: 'nav'
        }
    },
    data() {
        return {
            tab: 'headings',
            headings: [],
            navs: [],
            showNav: false,
            mouseover: false,
            intersectionObserver: null,
            mutationObserver: null,
            slotHeight,
            headingsSlots: Math.floor(window.innerHeight / slotHeight),
            navSlots: Math.floor(window.innerHeight / slotHeight)
        };
    },
    created() {
        window.addEventListener('scroll', this.onScroll,{ passive: true });
        window.addEventListener('mousemove', this.onMouseMove,{ passive: true });
        window.addEventListener('resize', this.onResize,{ passive: true });

        this.intersectionObserver = new IntersectionObserver(this.onIntersectionObserve);
        this.mutationObserver = new MutationObserver(this.onMutationObserve);
        this.mutationObserver.observe(document.body, {
            subtree: true,
            childList: true
        });
    },
    destroyed() {
        window.removeEventListener('scroll', this.onScroll);
        window.removeEventListener('mousemove', this.onMouseMove);
        window.removeEventListener('resize', this.onResize);

        this.intersectionObserver.disconnect();
        this.mutationObserver.disconnect();
    },
    mounted() {
        this.$refs.outline.addEventListener('mouseover', debounce(() => {
            this.mouseover = true;
        }, 100));

        this.$refs.outline.addEventListener('mouseleave', debounce(() => {
            this.mouseover = false;
        }, 1000));

        this.rebuildElements();
    },
    methods: {
        getTitleForElement(element) {
            const ariaLabel = element.getAttribute('aria-label');
            if (ariaLabel) {
                return ariaLabel;
            }

            const ariaLabelledBy = element.getAttribute('aria-labelledby');
            if (ariaLabelledBy) {
                const ariaLabeller = document.getElementById(ariaLabelledBy);

                if (ariaLabeller) {
                    return this.getTitleForElement(ariaLabeller);
                }
            }

            const title = element.getAttribute('title');
            if (title) {
                return title;
            }

            return element.textContent.substring(0, 100);
        },
        rebuildElements() {
            this.headings = [];
            const headings = document.querySelectorAll(this.headingsSelector);
            Array.from(headings).forEach(element => {
                this.intersectionObserver.observe(element);

                const title = this.getTitleForElement(element);
                const slot = Math.floor((element.getBoundingClientRect().top / document.body.clientHeight) * this.headingsSlots);

                this.headings[slot] = {
                    element,
                    title,
                    visible: false
                };
            });

            this.navs = [];
            const navs = document.querySelectorAll(this.navSelector);
            Array.from(navs).forEach(element => {
                this.intersectionObserver.observe(element);

                const title = this.getTitleForElement(element);
                const slot = Math.floor((element.getBoundingClientRect().top / document.body.clientHeight) * this.navSlots);

                this.navs[slot] = {
                    element,
                    title,
                    visible: false
                };
            });
        },
        onIntersectionObserve(entries) {
            entries.forEach(({ isIntersecting, target }) => {
                const header = this.headings.filter(v => v).find(node => node.element === target);
                if (header) {
                    header.visible = isIntersecting;
                }

                const nav = this.navs.filter(v => v).find(node => node.element === target);
                if (nav) {
                    nav.visible = isIntersecting;
                }
            });
        },
        onMutationObserve(entries) {
            const entry = entries.find(({ target }) => !this?.$refs?.outline?.contains(target) ?? false);
            if (entry) {
                this.rebuildElements();
            }
        },
        onScroll: throttle(function() {
            this.showNav = true;
            this.navHide();
        }, 100),
        onMouseMove: throttle(function({ pageX }) {
             if (pageX >= document.body.getBoundingClientRect().width) {
                 this.showNav = true;
                 this.navHide();
             }
        }, 100),
        onResize: debounce(function() {
            this.calculateSlots();
            this.rebuildElements();
        }, 500),
        onClick(element = null) {
            element?.scrollIntoView({ behavior: 'smooth' });
        },
        navHide: debounce(function() {
            this.showNav = false;
        }, 2000),
        calculateSlots() {
            this.headingsSlots = Math.floor(window.innerHeight / slotHeight);
            this.navSlots = Math.floor(window.innerHeight / slotHeight);
        },
        switchTab(tab) {
            this.tab = tab;
        },
        isTab(tab){
            return this.tab === tab;
        }
    },
    computed: {
        isVisible() {
            return this.mouseover || this.showNav;
        }
    }
}
</script>

<style scoped>
button {
    appearance: none;
    background: #fff;
    border-radius: 0;
    padding: 2px 5px;
    border: 1px solid gray;
}

.nav {
    position: absolute;
    transform: rotate(90deg);
    transform-origin: top left;

    opacity: 0;
    pointer-events: none;
    transition: opacity 200ms ease-in-out;
    will-change: opacity;
}

.nav button {
    cursor: pointer;
}

#outline.visible .nav {
    opacity: 1;
    pointer-events: initial;
}

button.visible {
    background: #000;
    color: #fff;
}

#outline {
    z-index: 99999;
    position: fixed;
    top: 0;
    right: 0;
    width: 1px;
    height: 100vh;

    margin: 0;
    padding: 0;

    background: #fff;
    border: none;
    box-shadow: -3px 0 20px 2px rgba(0,0,0,0.2);

    transition: width 200ms ease-in-out;
    will-change: width;
}

#outline.visible {
    width: 20%;
}

.tab {
    position: relative;
    top: 5px;
}

.node {
    display: block;
    box-sizing: border-box;
    width: 100%;
    height: calc(v-bind(slotHeight) * 1px);
    margin: 0;
    padding: 0 10px;
    font-size: 12px;
    line-height: 1;
    white-space: nowrap;
    text-overflow: ellipsis;
    overflow: hidden;
    color: rgba(0, 0,0, 0.8);
}

.node.enabled {
    cursor: pointer;
}

.node.visible {
    color: rgba(0, 0, 0, 1);
    font-weight: bold;
}
</style>
