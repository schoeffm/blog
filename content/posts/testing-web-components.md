+++
title = "Testing Web Components"
date = 2019-08-02T08:14:21+02:00
tags = ["web-components", "testing"]
+++

Testing web-components in isolation can be a challanging task. The (_current_) usual suspects in this area still [lack support for the standard](https://github.com/jsdom/jsdom/issues/1030) and thus cannot be used (at least at the time of this writing). Also, the web is swamped by posts about frameworks and thus finding good example projects or tutorials is also challanging.<br/>
Finally, we found, at least for our project, a good solution which does the job and whose setup I'd like to write down/preserve.
<!--more-->
So after a few flops we stumbled upon [the open-wc project][openwc] and their [testing modules][openwc-testing]. Since we had our project already setup we were just interested in the testing stuff - thus we tried to integrate only those dependencies (and skipped the scaffolding-, webpack-, transpiling-stuff).<br/>

## Setting things up

Let's start with a blank slate where we just install the [openwc-testing dependencies](https://open-wc.org/testing/testing-karma.html#default-configuration) (and their pre-requisites).

{{< highlight bash>}}
$ mkdir -p testing-wc/src/main/js && \
    cd testing-wc && \
    npm init -f
$ npm -i --save-dev @open-wc/testing-karma @open-wc/testing lit-html karma mocha 
{{< /highlight >}}

Next, let's create some components we can test - a parent-component which holds a child-component. The latter represents a simple UI-component while the parent also contains a `fetch`-call and thus interacts with the backend/network. The project structure looks like that:

{{< highlight bash>}}
.
├── karma.conf.js
├── package.json
└── src
    └── main
        ├── js
        │   ├── app.js
        │   ├── components
        │   │   ├── child.component.js
        │   │   └── parent.component.js
        │   └── index.html
        └── test
            ├── api
            │   └── customers
            └── components
                ├── child.component.test.js
                └── parent.component.test.js
{{< /highlight >}}

`index.html` and `app.js` are not relvant for this post (they only glue the components together). But before we can focus on the components and their tests we should have a look at `karma.conf.js` first.

Since we use [karma][karma] to launch and subsequently test our components we have to provide a suitable `karma.config.js` next to our `package.json`. Thanks to the [open-wc project][openwc] we can start using [their suggestion](https://open-wc.org/testing/testing-karma.html#manual-setup) and adjust it slightly to match our project-structure:

{{< highlight javascript >}}
const {createDefaultConfig} = require('@open-wc/testing-karma');
const merge = require('deepmerge');

module.exports = config => {
    config.set(
            merge(createDefaultConfig(config), {
                files: [
                    {pattern: './src/**/test/api/**/*', watched: true, included: false, served: true},
                    {pattern: config.grep ? config.grep : './src/main/test/**/*.test.js', type: 'module'}
                ],
            esm: {
                nodeResolve: true
            },
            proxies: {
                '/sfm/': '/base/src/main/test/'
            },
        }),
    );
    return config;
};
{{< /highlight >}}

Beside referencing our test descriptions under `src/main/test/**` we also let karma _serve_ files from the `src/**/test/api` (sub-)folders.<br/>
In addition to this we instruct karma to proxy requests to `/sfm/` and map 'em to the defined folder. The reason for this is explained later when we have to deal with API calls.

## Let's write some tests

After setting everything up we can finally have a look at the actual components and their tests. Let's start with the simple child-component first:

{{< highlight javascript>}}
export default class ChildComponent extends HTMLElement {
    constructor() {
        super();
        this.root = this.attachShadow({mode: 'open'});
    }

    set firstname(firstname) { this.setAttribute("firstname", firstname); }
    get firstname() { return this.getAttribute("firstname"); }

    set lastname(lastname) { this.setAttribute("lastname", lastname); }
    get lastname() { return this.getAttribute("lastname"); }

    async connectedCallback() {
        customElements
            .whenDefined('sfm-child')
            .then(_ => this.render());
    }

    render() {
        this.root.innerHTML = `
            <div>Hello ${this.firstname} ${this.lastname}</div>
        `;
    }
}
customElements.define('sfm-child', ChildComponent);
{{< /highlight >}}
This, quite simple, example should get us started.<br/> 
As you can see, it takes two properties and renders their content to shadow-dom. It doesn't define any logic which can be tested - so at least let's verify that it renders as expected:

{{< highlight javascript>}}
import {fixture, expect} from '@open-wc/testing';
import '../../js/components/child.component.js';

describe('Child-Component', () => {
    let underTest;
    beforeEach(async () => {
        underTest = await fixture('<sfm-child firstname="Luke" lastname="Skywalker"></sfm-child>');
    });
    describe('When initialized', () => {
        it('will render given properties', async () => {
            expect(underTest.firstname).to.be.eq('Luke');
            expect(underTest.lastname).to.be.eq('Skywalker');
            expect(underTest.shadowRoot.innerHTML).to.be.eq('<div>Hello Luke Skywalker</div>')
        });
    });
});
{{< /highlight >}}

First, we import two helper-methods from [open-wc testing][openwc-testing]. `fixture` renders our custom component (which gets imported in the next line) into the karma testpage and returns a reference to this dom element. All this is done in the `beforeEach`-block. `expect` opens the door to [chai][chai] (which is used under the hoods) and let us verify our expectations.<br/>
So here we just make sure that when we render our child-component with two pre-defined attribute values the component:

- sets the given attribues also as properties of the component
- outputs/renders HTML into shadow dom as expected

Although it is dead simple this test should get you started. Since you got a reference to the component you can interact with its JS-API to change its state and verify that change using `expect`. 

## Test with network dependency

To make things a little bit more complicated let's add an API roundtrip to the game. The `ParentComponent` embeds the `ChildComponent` and updates its properties with data `fetch`ed from the backend.<br/>
In order to simulate that we've setup karma to proxy all requests to `/sfm/*` and map those requests to static API-fixtures. So for example a request to `/sfm/api/customers` will be mapped to `./src/main/test/api/customers`. Let's create that file quickly so that karma can actually server content from there:

{{< highlight bash >}}
$ mkdir -p ./src/main/test/api/customers && \
    echo -n '{"id":4711,"firstname":"johann","lastname":"doe"}' > ./src/main/test/api/customers 
{{< /highlight >}}

Next, let's write the actual component:

{{< highlight javascript>}}
import ChildComponent from './child.component.js';

export default class ParentComponent extends HTMLElement {

    constructor() {
        super();
        this.root = this.attachShadow({mode: 'open'});
    }

    connectedCallback() {
        customElements
            .whenDefined('sfm-parent')
            .then(() => this.render())
            .then(() => this.root.querySelector('sfm-child'))
            .then((node) => this._loadData(node));
    }

    render() {
        this.root.innerHTML = `<sfm-child></sfm-child>`;
    }

    _loadData(node) {
        fetch('/sfm/api/customers')
            .then(res => res.json())
            .then(({firstname, lastname}) => {
                node.firstname = firstname;
                node.lastname = lastname;
            });
    }
}
customElements.define('sfm-parent', ParentComponent);
{{< /highlight >}}

The component renders the `ChildComponent` _without any attributes_. As soon as it gets connected it'll load data from `/sfm/api/customers` in order to set the result as props to the embedded `ChildComponent`. Let's see how to deal with that in our test:

{{< highlight javascript >}}
import { fixture, expect } from '@open-wc/testing';
import { aTimeout } from '@open-wc/testing-helpers';

import '../../js/components/parent.component.js';

describe('Parent-Component', () => {
    let underTest;
    beforeEach(async () => {
        underTest = await fixture('<sfm-parent></sfm-parent>');
    });

    describe('When initialized', () => {
        it('will render, fetch data and update child-component', async () => {
            await aTimeout(100); // wait for the network call

            const child = underTest.shadowRoot.querySelector('sfm-child');
            expect(child.firstname).to.be.eq('johann');
            expect(child.lastname).to.be.eq('doe');
        });
    });
});
{{< /highlight >}}

As before, using the `fixture`-method from [openwc][openwc-testing] we render our `ParentComponent` - when that happens the component issues the network call and in order to wait for the result to arrive we have to make use of the `aTimeout` helper method. Since this has nothing to do with rendering- or the component-lifecycle itself there is no elegant way to wait for the network roundtrip as to just pause a couple of milliseconds.<br/>
After that we can check the props of the embedded component if they received our remote data and thus if the plumbing done by `ParentComponent` worked.


[rollup]:http://rollupjs.org/
[openwc-testing]:https://open-wc.org/testing/
[openwc]:https://open-wc.org/
[puppeteer]:https://pptr.dev/
[karma]:http://karma-runner.github.io/latest/index.html
[mocha]:https://mochajs.org/
[chai]:https://www.chaijs.com/api/
[repo]:https://github.com/schoeffm/testing-web-components
