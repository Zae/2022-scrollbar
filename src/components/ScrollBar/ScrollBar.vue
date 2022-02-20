<template>
    <aside
        id="outline"
        ref="outline"
        :class="{ visible: isVisible }"
    >
        <section class="nav">
            <button :class="{ visible: this.tab === 'headings' }" @click="this.tab = 'headings'">HEADINGS</button>
            <button :class="{ visible: this.tab === 'navs' }" @click="this.tab = 'navs'">NAVS</button>
        </section>
        <section v-if="tab === 'headings'" data-tab="headings">
            <span
                v-for="[element, node] in headings"
                :key="element"
                v-text="node.title"
                class="node"
                :class="{ visible: node.visible }"
                :style="`top: ${node.pagePosition}%`"
                @click="onClick(element)"
            />
        </section>
        <section v-if="tab === 'navs'" data-tab="nav">
            <span
                v-for="[element, node] in navs"
                :key="element"
                v-text="node.title"
                class="node"
                :class="{ visible: node.visible }"
                :style="`top: ${node.pagePosition}%`"
                @click="onClick(element)"
            />
        </section>
    </aside>
</template>

<script>
import { debounce, throttle } from 'lodash';

export default {
    data() {
        return {
            tab: 'headings',
            headings: null,
            navs: null,
            showNav: false,
            mouseover: false,
            intersectionObserver: null,
            mutationObserver: null
        };
    },
    created() {
        window.addEventListener('scroll', this.onScroll,{
            passive: true
        });
        window.addEventListener('mousemove', this.onMouseMove,{
            passive: true
        });

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
        this.intersectionObserver.disconnect();
        this.mutationObserver.disconnect();
    },
    mounted() {
        this.$refs.outline.addEventListener('mouseover', debounce(() => {
            this.mouseover = true;
        }), 100);

        this.$refs.outline.addEventListener('mouseleave', debounce(() => {
            this.mouseover = false;
        }), 1000);

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
            const headings = document.querySelectorAll('h1,h2,h3,h4,h5,h6');

            this.headings = new Map();
            Array.from(headings).forEach(element => {
                this.intersectionObserver.observe(element);

                const title = this.getTitleForElement(element);
                const position = element.getBoundingClientRect().top + window.scrollY;

                this.headings.set(element, {
                    element,
                    title,
                    position,
                    pagePosition: (position / (document.body.clientHeight * 1.025) * 100) + 0.8,
                    visible: false
                });
            });

            const navs = document.querySelectorAll('nav');

            this.navs = new Map();
            Array.from(navs).forEach(element => {
                this.intersectionObserver.observe(element);

                const title = this.getTitleForElement(element);
                const position = element.getBoundingClientRect().top + window.scrollY;

                this.navs.set(element, {
                    element,
                    title,
                    position,
                    pagePosition: (position / (document.body.clientHeight * 1.025) * 100) + 0.8,
                    visible: false
                });
            });
        },
        onIntersectionObserve(entries) {
            entries.forEach(({ isIntersecting, target }) => {
                const heading = this.headings.get(target);
                if (heading) {
                    heading.visible = isIntersecting;
                }

                const nav = this.navs.get(target);
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
        onClick(element) {
            element.scrollIntoView({ behavior: 'smooth' });
        },
        navHide: debounce(function() {
            this.showNav = false;
        }, 2000)
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
    -webkit-appearance: none;
    background: #fff;
    border-radius: 0;
    padding: 2px 5px;
    border: 1px solid gray;
}

.nav {
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

.node {
    box-sizing: border-box;
    position: absolute;
    font-size: 12px;
    line-height: 1;
    margin: 0;
    padding: 0 10px;
    white-space: nowrap;
    text-overflow: ellipsis;
    overflow: hidden;

    width: 100%;

    display: inline-block;
    /*background-color: white;*/

    color: rgba(0,0,0, 0.8);
    cursor: pointer;
}

.node.visible {
    color: rgba(0,0,0, 1);
    font-weight: bold;
}
</style>
