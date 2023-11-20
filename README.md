<h2 align="center">Integration testing suit for Rails Stimulus and View Partials</h2>

**👩🏻‍💻 Developer Ready**: A comprehensive integration testing suite utilizing JEST and a static HTML generator.

**📸 Scalable testing**: This suite focuses on testing JavaScript and views without creating database records, providing a scalable solution.

**🏃🏽 Instant Feedback**: Easily compare changes and build/test only modified files for efficient development.

## The Problem

Consider the typical test case below, which offers limited value as the innerHTML is hardcoded and not based on the actual server-side HTML. When the view changes on the server side, the test case may not fail.
```javascript
it('should render enabled staff', async () => {
  const div = document.createElement('div');
  div.innerHTML = '<button class="selectable">Select Me</button>';
  document.body.appendChild(div);

  const button = screen.getByClass('selectable');

  expect(button).toBeInTheDocument();
  await fireEvent.click(button);
  expect(button.getAttribute('class')).toContain('selected');
});
```

## The solution

```ruby
RailsJest.scope '/example' do
  @button_text = 'Select Me'
  render partial: 'example/button'
end
```

A static HTML file, example.html, is generated. The JEST test case is then executed against this generated HTML. If the view changes, the static HTML is regenerated for the corresponding test case.

## Getting Started

<!-- copied from Getting Started docs, links updated to point to Jest website -->

Install stimulus-jest using [`gem`](https://github.com/rubygems/rubygems):

```bash
gem install --dev stimulus-jest
```

Install Jest using [`yarn`](https://yarnpkg.com/en/package/jest):

```bash
yarn add --dev jest
```

Or [`npm`](https://www.npmjs.com/package/jest):

```bash
npm install --save-dev jest
```

## Setup
```ruby
RailsJest.configure do |config|
  # Path to store the ruby file to generate snapshot
  # default: 'spec/support/generators'
  config.factory_path = 'spec/support/generators'

  # Path to store html snapshots
  # default: 'spec/fixture/snapshots'
  config.snapshot_path = 'spec/fixture/snapshots'
end
```

## Example
Let's get started by creating a snapshot generators `spec/support/generators/staff.generator.rb`

```ruby
RailsJest.scope '/admin/staffs' do
  @staffs = FactoryBot.build_list(:staff, 5, :with_employment_date)

  RailsJest.define '/admin/staffs/table' do
    render partial: 'admin/staffs/table'
  end

  RailsJest.define '/admin/staffs/[0-9]*/toggle' do
    staffs.first.status = :disable

    respond_to do |format|
      format.html { redirect_to admin_staff_path }
      format.turbo_stream do
        render turbo_stream: turbo_stream.replace_all('staff_table', partial: 'admin/staff/table')
      end
    end
  end
end
```

This will generate 3 html files.
1. `spec/fixture/snapshots/admin/staffs/table`
2. `spec/fixture/snapshots/admin/staffs/[0-9]*/toggle.html`
3. `spec/fixture/snapshots/admin/staffs/[0-9]*/toggle.turbo_stream`

When performing JS-DOM testing:

```javascript
describe('admin / staff', () => {
  testDom(['/admin/staffs/table'], () => {
    it('xxx', () => {
      ...
    });
  });
...
```

It loads the HTML file spec/fixture/snapshots/admin/staffs/table for the JS-DOM testing.


## Working with turbo stream

When a request is made to turbo-stream, "@rails/request.js" is mocked to return the file that first regex matches the file path. e.g. `selectable_table_controller.js`

```javascript
import { Controller } from "@hotwired/stimulus"
import { get } from "@rails/request.js"

export default class extends Controller {
   toggle() {
    get('/admin/staffs/1/toggle', {
      responseKind: "turbo-stream",
      query: formData,
    });
   }
}
```

get('/admin/staffs/1/toggle') will return the html file of '/admin/staffs/[0-9]*/toggle.turbo_stream'.

## Full example

Consider the view that generates a selectable table. First, create the partial _table.html.slim:

```slim
= selectable_table('staff_table') do
  tr
    th = t('tables.header.name')
    th = t('tables.header.staff_number')
    th = t('tables.header.staff_employment_date')
  = render collection: @staffs, partial: 'staff'
```

The partial `_staff.html.slim`
```slim
= selectable_row, class: staff.enable? ? 'enabled' : 'disabled' do
  td = staff.name
  td = staff.number
  td = staff.employment_date
```

The view helper `selectable_table.rb`
```ruby
def selectable_table(id = 'table', &block)
  content_tag :table, id:, data: { controller: 'selectable-table' } do
    catpure(&block) if block_given?
  end
end

def selectable_row(**opts, &)
  content_tag :tr, data: { action: 'selectable-table#toggle' }, **opts do
    catpure(&block) if block_given?
  end
end
```

Then, create a file named `selectable-table.spec.js`. This will contain our actual test:

```javascript
import { screen, fireEvent, waitFor } from '@testing-library/dom';
import { Application } from '@hotwired/stimulus';
import SelectableTableController from 'controllers/selectable_table_controller';
import { testDom } from 'stimulus-jest';

const startStimulus = () => {
  const application = Application.start();
  application.register('selectable-table', SelectableTableController);
};

describe('admin/staff', () => {
  beforeEach(() => {
     startStimulus();
  });

  testDom(['/admin/staffs/table'], () => {
    it('should render enabled staff', async () => {
      const row = screen.getByClass('enable');
  
      expect(row).toBeInTheDocument();
      await fireEvent.click(a1);
      expect(a1.getAttribute('class')).toContain('selected');
    });
  
    it('should be able to toggle staff', async () => {
      const row = screen.findFirstByTagName('tr');
      await row.click(a1);
      expect(row.getAttribute('class').toContain('disabled');
    });
  });
});
```
