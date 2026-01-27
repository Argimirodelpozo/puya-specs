# Infinite loop detection

```py
class NoInfiniteLoopsValidator(DestructuredIRValidator):
    @typing.override
    def visit_block(self, block: models.BasicBlock) -> None:
        assert block.terminator is not None, "unterminated block found during IR validation"
        if block.terminator.unique_targets == [block]:
            logger.error(
                "infinite loop detected",
                location=block.terminator.source_location or block.source_location,
            )
```
This validation pass checks that each basic block has a terminator, and that the target block for the terminator is _not_ uniquely itself, as that would constitute a very simple case of an infinite loop.
> [!Note]
> Infinite loops don't make sense in the AVM architecture, as they would just consistently run out of budget, reverting any state changes they may attempt and wasting computation.
<!-- TODO: what about an use case for programs not meant to be run but only simulated (e.g. Algoland Tasos example) -->